Sync HubSpot Contacts with LinkedIn URLs, Detect Changes & Send Gmail Alerts

https://n8nworkflows.xyz/workflows/sync-hubspot-contacts-with-linkedin-urls--detect-changes---send-gmail-alerts-3923


# Sync HubSpot Contacts with LinkedIn URLs, Detect Changes & Send Gmail Alerts

### 1. Workflow Overview

This n8n workflow automates the synchronization of HubSpot contacts enriched with LinkedIn profile URLs, monitors changes in their LinkedIn posts or professional positions, and sends alert emails via Gmail when updates occur. It is designed for customer success managers, sales, and marketing teams who want to maintain an up-to-date contact database with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Settings Initialization**: Setup of email, Google Sheet link, and API credentials used throughout the workflow.
- **1.2 HubSpot API Connection & Owner Identification**: Fetching HubSpot contact owners and filtering the current owner running the workflow.
- **1.3 Client Fetching with Pagination**: Retrieving all contacts (clients) associated with the current owner, handling API pagination.
- **1.4 LinkedIn URL Enrichment**: Searching LinkedIn profiles first by name and company, then by profile URL if missing; merging profile URLs into contact records.
- **1.5 Profile Update Detection**: Checking for updates in LinkedIn posts and positions by comparing with stored data.
- **1.6 Data Storage in Google Sheets**: Storing and updating contact info, LinkedIn URLs, last posts, and positions for comparison.
- **1.7 Email Alert Generation and Sending**: Compiling alert emails listing contacts with updates and sending them through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Settings Initialization

**Overview:**  
Initial configuration inputs including owner email for filtering, Google Sheet URL for data storage, and email address for alert sending.

**Nodes Involved:**  
- Set data here  
- Sticky Note1  
- Sticky Note  
- Sticky Note8 (Contact info)

**Node Details:**

- **Set data here**  
  - Type: Set node  
  - Role: Defines workflow-level constants: owner email ("zeerobug@gmail.com") and Google Sheet URL.  
  - Key Variables: `email`, `sheetLink`  
  - Input: Manual trigger or upstream node  
  - Output: Provides these values to downstream nodes via expressions  
  - Edge Cases: Misconfiguration leads to filtering or storage errors.

- **Sticky Notes**  
  - Provide instructions and documentation about settings and contact for support.  
  - Do not affect workflow logic.

---

#### 2.2 HubSpot API Connection & Owner Identification

**Overview:**  
Fetches the list of HubSpot contact owners and filters to identify the current workflow executor (owner).

**Nodes Involved:**  
- When clicking ‘Test workflow’ (manual trigger)  
- Get list of owners (HTTP Request)  
- Split Out owners (Split Out)  
- Get current owner (Filter)  
- Sticky Note (HubSpot API instructions)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually.  
  - No inputs; outputs trigger next nodes.

- **Get list of owners**  
  - Type: HTTP Request  
  - Role: Calls HubSpot API endpoint to get owners (users with contact access).  
  - Config: Uses OAuth2 Credential for HubSpot.  
  - URL: `https://api.hubapi.com/crm/v3/owners`  
  - Outputs a list of owners for further processing.  
  - Failure Modes: OAuth token expiry, API rate limits, connectivity issues.

- **Split Out owners**  
  - Type: Split Out  
  - Role: Splits list of owners into individual items for filtering.  
  - Input: List of owners from API.  
  - Output: Individual owner records.

- **Get current owner**  
  - Type: Filter  
  - Role: Filters owners to find the one matching the configured email in "Set data here".  
  - Condition: Match owner email with `email` from settings node.  
  - Output: The owner record used to fetch associated clients.  
  - Edge Cases: If email not found, no clients fetched.

- **Sticky Note**  
  - Provides instructions and link to HubSpot OAuth2 credential setup.  
  - Reference: https://docs.n8n.io/integrations/builtin/credentials/hubspot/

---

#### 2.3 Client Fetching with Pagination

**Overview:**  
Fetches all contacts (clients) for the current owner, handling pagination due to HubSpot API limits (200 contacts per page).

**Nodes Involved:**  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Edit (Set)  
- Get list of clients for owner (HTTP Request)  
- Increment Page (Set)  
- If (Conditional)  
- Merge al the entries (Code)  
- Split Out1 (Split Out)  
- Sticky Note10 (Pagination explanation)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Enables the sub-workflow to be callable with `ownerId` parameter.  
  - Input: `ownerId` from parent workflow.  
  - Output: Triggers client fetching process.

- **Edit**  
  - Type: Set  
  - Role: Initializes pagination variables: `sofar` (offset) = 0, `results` = empty array.  
  - Output: Initial state for pagination loop.

- **Get list of clients for owner**  
  - Type: HTTP Request  
  - Role: Calls HubSpot contacts search API filtered by owner ID, fetching contacts with properties firstname, lastname, email, linkedinURL, company.  
  - Method: POST  
  - Pagination: Uses `sofar` as offset with `after` parameter for pagination.  
  - Outputs contacts for the current page.

- **Increment Page**  
  - Type: Set  
  - Role: Updates `sofar` by adding the number of contacts fetched in the current batch.  
  - Executes once per pagination cycle to update offset.

- **If**  
  - Type: Conditional  
  - Role: Checks if pagination should continue by comparing `sofar` offset with total contacts available.  
  - Condition: Continue if `sofar` < total contacts count.

- **Merge al the entries**  
  - Type: Code  
  - Role: Attempts to merge all paginated results into a single array (note: code uses a loop with try-catch to collect all pages).  
  - Edge Cases: Potential infinite loop if API response structure changes or errors occur.

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits merged contacts array into individual contact items for processing.

- **Sticky Note10**  
  - Explains API pagination limitation and necessity of loop for all contacts.

---

#### 2.4 LinkedIn URL Enrichment

**Overview:**  
For each contact, attempts to find and enrich LinkedIn profile URLs by searching LinkedIn via RapidAPI services using contact names and company info. If no URL is found, a secondary search by profile link is performed, merging the URLs into records.

**Nodes Involved:**  
- Change this for testing (Filter)  
- Create entry with email (Google Sheets)  
- Get rows from document (Google Sheets)  
- If linkedin url is empty (If)  
- Search for user profile by names (HTTP Request)  
- Profile URL not found? (If)  
- Do nothing (NoOp)  
- Set the profile URL (Set)  
- Set the profile URL1 (Set)  
- Merge profileURL (Code)  
- Search for user by link (HTTP Request)  
- Sticky Note6 (LinkedIn search instructions)  
- Sticky Note3 (Testing filter instructions)  
- Sticky Note4 (Google Sheet preparation instructions)

**Node Details:**

- **Change this for testing**  
  - Type: Filter  
  - Role: Filters contacts to a smaller subset for testing (e.g., by specific email).  
  - Configurable to limit workload during tests.

- **Create entry with email**  
  - Type: Google Sheets  
  - Role: Appends or updates a row for each contact's email in the specified Google Sheet for tracking.  
  - Mapping: Uses email field.  
  - Credential: Google Sheets OAuth2.

- **Get rows from document**  
  - Type: Google Sheets  
  - Role: Retrieves existing rows matching the contact email for comparison.  
  - Filters by email to fetch specific contact data from sheet.

- **If linkedin url is empty**  
  - Type: Conditional  
  - Role: Checks if LinkedIn URL exists for the contact.  
  - Condition: Empty LinkedIn URL triggers secondary search.

- **Search for user profile by names**  
  - Type: HTTP Request  
  - Role: Queries RapidAPI LinkedIn people search API using firstname, lastname, and company.  
  - Authentication: RapidAPI key via HTTP header.  
  - Outputs profile data with possible LinkedIn URL.

- **Profile URL not found?**  
  - Type: Conditional  
  - Role: Checks if search returned a profile URL.  
  - If empty, bypasses update (NoOp); else sets the profile URL.

- **Do nothing**  
  - Type: No Operation node  
  - Role: Placeholder to skip processing if no URL found.

- **Set the profile URL**  
  - Type: Set  
  - Role: Extracts and sets the profile URL from API response.

- **Set the profile URL1**  
  - Type: Set  
  - Role: Sets the profile URL from Google Sheet if already existing.

- **Merge profileURL**  
  - Type: Code  
  - Role: Merges extracted profile URL into contact item JSON for downstream use.

- **Search for user by link**  
  - Type: HTTP Request  
  - Role: Retrieves detailed LinkedIn profile data using the profile URL via RapidAPI.  
  - Outputs last post text and position data.

- **Sticky Notes**  
  - Provide contextual instructions about LinkedIn API key setup and Google Sheet preparation.

---

#### 2.5 Profile Update Detection

**Overview:**  
Compares the latest LinkedIn post and position information with stored data to detect updates, then updates Google Sheets accordingly and sets flags for sending alerts.

**Nodes Involved:**  
- Get last post (Execute Workflow)  
- Set last_post (Set)  
- if new post (If)  
- Update last post (Google Sheets)  
- Set post_updated (Set)  
- Set last_position (Set)  
- if new position (If)  
- Updates last position (Google Sheets)  
- Set position_updated (Set)  
- Sticky Note5 (Additional tests instructions)

**Node Details:**

- **Get last post**  
  - Type: Execute Workflow  
  - Role: Calls a separate workflow to fetch the latest LinkedIn posts for the user.  
  - Inputs: username, maxItems=1, responseType=detail.  
  - Output: Latest post data.

- **Set last_post**  
  - Type: Set  
  - Role: Sets the last post text from fetched data as `last_post`.

- **if new post**  
  - Type: Conditional  
  - Role: Compares new post text with stored post text in Google Sheets.  
  - If different, triggers update.

- **Update last post**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet with new post data, LinkedIn URL, position, and current date.

- **Set post_updated**  
  - Type: Set  
  - Role: Flags the contact as having a post update and includes the email for alerting.

- **Set last_position**  
  - Type: Set  
  - Role: Sets last known position title from LinkedIn data.

- **if new position**  
  - Type: Conditional  
  - Role: Compares new position title with stored position from Google Sheets.

- **Updates last position**  
  - Type: Google Sheets  
  - Role: Updates Google Sheet with new position and related info.

- **Set position_updated**  
  - Type: Set  
  - Role: Flags the contact as having a position update with email.

- **Sticky Note5**  
  - Notes on potential for adding more update tests, e.g., new LinkedIn comments, HubSpot activities.

---

#### 2.6 Email Alert Generation and Sending

**Overview:**  
Generates an email message listing all contacts with updated posts or positions and sends it to configured email(s) via Gmail.

**Nodes Involved:**  
- Merge on email (Merge)  
- Generate the email text (Code)  
- Gmail (Send email)  
- Sticky Note7 (Email alert instructions)

**Node Details:**

- **Merge on email**  
  - Type: Merge  
  - Role: Combines input streams (post updates and position updates) based on email, keeping all entries.

- **Generate the email text**  
  - Type: Code  
  - Role: Aggregates emails flagged with post or position updates, constructs a human-readable alert text summarizing changes.  
  - Logic:  
    - Creates arrays for clients with post updates and position updates.  
    - Builds message sections if updates exist.

- **Gmail**  
  - Type: Gmail node  
  - Role: Sends alert email with generated text as message body.  
  - Config: Uses Gmail OAuth2 credential; recipient email from settings node.  
  - Email type: Plain text; subject "Changes in your clients".  
  - Possible Failures: Authentication errors, quota limits.

- **Sticky Note7**  
  - Describes email alert generation block.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                                    | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                                       |
|----------------------------|---------------------------|---------------------------------------------------|----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger            | Starts the workflow manually                       |                                  | Set data here                      |                                                                                                                                  |
| Set data here              | Set                       | Initializes settings: owner email and sheet link  | When clicking ‘Test workflow’    | Get list of owners                 | Settings: set your email and Google Sheet link. Link to example sheet provided.                                                  |
| Get list of owners         | HTTP Request              | Fetches HubSpot contact owners                     | Set data here                   | Split Out owners                  | HubSpot API setup instructions with link.                                                                                        |
| Split Out owners           | Split Out                 | Splits owners list into individual records        | Get list of owners               | Get current owner                 |                                                                                                                                  |
| Get current owner          | Filter                    | Filters owner matching configured email            | Split Out owners                | Get list of clients               |                                                                                                                                  |
| When Executed by Another Workflow | Execute Workflow Trigger | Sub-workflow trigger with ownerId parameter       | Get current owner               | Edit                            |                                                                                                                                  |
| Edit                      | Set                       | Initializes pagination variables                   | When Executed by Another Workflow| Get list of clients for owner    |                                                                                                                                  |
| Get list of clients for owner | HTTP Request            | Fetches contacts for owner with pagination        | Edit                           | Increment Page                   | Pagination explanation note.                                                                                                    |
| Increment Page            | Set                       | Updates pagination offset                           | Get list of clients for owner   | If                             |                                                                                                                                  |
| If                        | Conditional               | Checks whether more pages to fetch                 | Increment Page                  | Get list of clients for owner / Merge al the entries |                                                                                                                                  |
| Merge al the entries       | Code                      | Merges paginated contact lists                      | If                            | Split Out1                     |                                                                                                                                  |
| Split Out1                 | Split Out                 | Splits merged contacts into individual items       | Merge al the entries            | Change this for testing          |                                                                                                                                  |
| Change this for testing    | Filter                    | Filters contacts for testing purposes               | Split Out1                    | Create entry with email          | Testing note recommending filtering a small number of clients.                                                                  |
| Create entry with email    | Google Sheets             | Appends/updates email entry in Google Sheet        | Change this for testing         | Get rows from document           | Instructions on Google Sheet preparation.                                                                                       |
| Get rows from document     | Google Sheets             | Retrieves contact's row from Google Sheet          | Create entry with email         | If linkedin url is empty         |                                                                                                                                  |
| If linkedin url is empty   | Conditional               | Checks if LinkedIn URL is missing                   | Get rows from document          | Search for user profile by names / Set the profile URL1 | LinkedIn URL search instructions.                                                                                              |
| Search for user profile by names | HTTP Request         | Searches LinkedIn profile by name & company        | If linkedin url is empty        | Profile URL not found?           |                                                                                                                                  |
| Profile URL not found?     | Conditional               | Checks if profile URL was found                      | Search for user profile by names| Do nothing / Set the profile URL |                                                                                                                                  |
| Do nothing                | NoOp                      | Skips processing if no profile URL found            | Profile URL not found?          |                                |                                                                                                                                  |
| Set the profile URL       | Set                       | Sets profile URL from API response                   | Profile URL not found?          | Merge profileURL                |                                                                                                                                  |
| Set the profile URL1       | Set                       | Sets profile URL from Google Sheet                   | If linkedin url is empty        | Merge profileURL                |                                                                                                                                  |
| Merge profileURL           | Code                      | Adds profile URL to contact item JSON                | Set the profile URL / Set the profile URL1 | Search for user by link        | LinkedIn search API key note.                                                                                                   |
| Search for user by link    | HTTP Request              | Gets detailed LinkedIn profile data by URL          | Merge profileURL               | Get last post / Set last_position |                                                                                                                                  |
| Get last post             | Execute Workflow          | Fetches latest LinkedIn post for user                | Search for user by link         | Set last_post                  | Additional tests instructions note.                                                                                            |
| Set last_post             | Set                       | Sets last post text for comparison                    | Get last post                  | if new post                   |                                                                                                                                  |
| if new post               | Conditional               | Detects if post has changed                            | Set last_post                  | Update last post / (no update)  |                                                                                                                                  |
| Update last post          | Google Sheets             | Updates Google Sheet with new post info                | if new post                   | Set post_updated               |                                                                                                                                  |
| Set post_updated          | Set                       | Flags contact as having a post update                  | Update last post              | Merge on email                 |                                                                                                                                  |
| Set last_position         | Set                       | Sets last position title from profile data             | Search for user by link         | if new position               |                                                                                                                                  |
| if new position           | Conditional               | Detects if position has changed                          | Set last_position             | Updates last position / (no update) |                                                                                                                                  |
| Updates last position     | Google Sheets             | Updates Google Sheet with new position info             | if new position               | Set position_updated           |                                                                                                                                  |
| Set position_updated      | Set                       | Flags contact as having a position update                 | Updates last position         | Merge on email                 |                                                                                                                                  |
| Merge on email            | Merge                     | Combines post and position update flags by email        | Set post_updated / Set position_updated | Generate the email text       | Email alert generation note.                                                                                                   |
| Generate the email text   | Code                      | Compiles alert text summarizing updates                 | Merge on email                | Gmail                        |                                                                                                                                  |
| Gmail                    | Gmail                     | Sends alert email with updates                            | Generate the email text       |                              | Gmail integration requires OAuth2 credential setup.                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: To manually start the workflow.

2. **Add a Set Node for Configuration**  
   - Name: "Set data here"  
   - Fields:  
     - `email`: Your HubSpot owner email (e.g., "zeerobug@gmail.com")  
     - `sheetLink`: URL of your Google Sheet for data storage (e.g., the provided example sheet)  
   - Connect from Manual Trigger.

3. **Add HTTP Request Node to Get HubSpot Owners**  
   - Name: "Get list of owners"  
   - Method: GET  
   - URL: `https://api.hubapi.com/crm/v3/owners`  
   - Authentication: Use HubSpot OAuth2 API credential.  
   - Connect from "Set data here".

4. **Add Split Out Node to Split Owners List**  
   - Name: "Split Out owners"  
   - Field to split: `results` or as per API response.  
   - Connect from "Get list of owners".

5. **Add Filter Node to Identify Current Owner**  
   - Name: "Get current owner"  
   - Condition: owner email equals expression from "Set data here" node's `email` field.  
   - Connect from "Split Out owners".

6. **Add Execute Workflow Trigger Node for Sub-workflow**  
   - Name: "When Executed by Another Workflow"  
   - Input schema: `ownerId` string parameter  
   - Connect from "Get current owner".

7. **Create a Sub-workflow for Client Fetching and Processing** (or continue in same workflow):

   - **Add Set Node**  
     - Name: "Edit"  
     - Initialize variables:  
       - `sofar` = 0  
       - `results` = []  
     - Connect from "When Executed by Another Workflow".

   - **Add HTTP Request Node to Fetch Clients**  
     - Name: "Get list of clients for owner"  
     - Method: POST  
     - URL: `https://api.hubapi.com/crm/v3/objects/contacts/search`  
     - Body: JSON with filter for `hubspot_owner_id` equals to `ownerId` parameter, properties: firstname, lastname, email, linkedinURL, company.  
     - Pagination with `after` equals `sofar`.  
     - Authentication: HubSpot OAuth2 API.  
     - Connect from "Edit".

   - **Add Set Node to Increment Pagination**  
     - Name: "Increment Page"  
     - Update `sofar` by adding the length of current page results.  
     - Connect from "Get list of clients for owner".

   - **Add Conditional Node to Check Pagination End**  
     - Name: "If"  
     - Condition: Continue if `sofar` < total contacts count.  
     - Connect from "Increment Page".  
     - Loop back to "Get list of clients for owner" if true; else continue.

   - **Add Code Node to Merge All Clients**  
     - Name: "Merge al the entries"  
     - Logic: Concatenate all pages’ results into one array.  
     - Connect from "If" node false branch.

   - **Add Split Out Node**  
     - Name: "Split Out1"  
     - Split merged clients into individual items.  
     - Connect from "Merge al the entries".

8. **Add Filter Node for Testing (Optional)**  
   - Name: "Change this for testing"  
   - Filter clients by email or other criteria for limited testing.  
   - Connect from "Split Out1".

9. **Add Google Sheets Node to Append/Update Contact Email**  
   - Name: "Create entry with email"  
   - Operation: appendOrUpdate  
   - Map `email` from client data.  
   - Authenticate with Google Sheets OAuth2.  
   - Connect from "Change this for testing".

10. **Add Google Sheets Node to Retrieve Existing Contact Rows**  
    - Name: "Get rows from document"  
    - Filter by email column equals client email.  
    - Connect from "Create entry with email".

11. **Add Conditional Node to Check if LinkedIn URL is Empty**  
    - Name: "If linkedin url is empty"  
    - Condition: Check if stored LinkedIn URL field is empty.  
    - Connect from "Get rows from document".

12. **Add HTTP Request Node to Search LinkedIn by Name and Company**  
    - Name: "Search for user profile by names"  
    - URL: RapidAPI LinkedIn people search endpoint  
    - Query Parameters: firstname, lastname, company from client data  
    - Authentication: RapidAPI HTTP Header Auth with your API key  
    - Connect from "If linkedin url is empty" true branch.

13. **Add Conditional Node to Check if Profile URL Found**  
    - Name: "Profile URL not found?"  
    - Condition: Check if profileURL in API response is empty.  
    - Connect from "Search for user profile by names".

14. **Add NoOp Node for Handling No URL Found**  
    - Name: "Do nothing"  
    - Connect from "Profile URL not found?" true branch.

15. **Add Set Node to Assign Profile URL from API Response**  
    - Name: "Set the profile URL"  
    - Assign profileURL field from API data.  
    - Connect from "Profile URL not found?" false branch.

16. **Add Set Node to Assign Profile URL from Google Sheet**  
    - Name: "Set the profile URL1"  
    - Assign profileURL from Google Sheet data (if exists).  
    - Connect from "If linkedin url is empty" false branch.

17. **Add Code Node to Merge Profile URL into Contact Data**  
    - Name: "Merge profileURL"  
    - Add/overwrite `profileURL` property in client JSON with value set previously.  
    - Connect from "Set the profile URL" and "Set the profile URL1".

18. **Add HTTP Request Node to Get LinkedIn Profile Details by URL**  
    - Name: "Search for user by link"  
    - URL: RapidAPI LinkedIn profile data endpoint  
    - Query Parameter: profileURL  
    - Authentication: RapidAPI HTTP Header Auth  
    - Connect from "Merge profileURL".

19. **Add Execute Workflow Node to Get Latest LinkedIn Post**  
    - Name: "Get last post"  
    - Inputs: username from LinkedIn profile data, maxItems=1, responseType=detail.  
    - Connect from "Search for user by link".

20. **Add Set Nodes to Store Last Post and Last Position**  
    - Names: "Set last_post" and "Set last_position"  
    - Extract last post text and current position title from LinkedIn profile data.  
    - Connect from "Get last post" and "Search for user by link" respectively.

21. **Add Conditional Nodes to Detect Changes**  
    - Names: "if new post" and "if new position"  
    - Compare fetched last post and position with stored values from Google Sheets.  
    - Connect from "Set last_post" and "Set last_position".

22. **Add Google Sheets Nodes to Update Last Post and Position**  
    - Names: "Update last post" and "Updates last position"  
    - Append or update records with new post, position, LinkedIn URL, and date.  
    - Connect from true branches of change detection nodes.

23. **Add Set Nodes to Flag Updates**  
    - Names: "Set post_updated" and "Set position_updated"  
    - Flags to true along with contact email for downstream merging.  
    - Connect from update Google Sheets nodes.

24. **Add Merge Node to Combine Update Flags by Email**  
    - Name: "Merge on email"  
    - Join mode: Keep everything, matching on email.  
    - Connect from "Set post_updated" and "Set position_updated".

25. **Add Code Node to Generate Email Text**  
    - Name: "Generate the email text"  
    - Logic: Aggregate emails flagged for post or position updates and build text summary.  
    - Connect from "Merge on email".

26. **Add Gmail Node to Send Alert Email**  
    - Name: "Gmail"  
    - Recipient: Email from settings node  
    - Subject: "Changes in your clients"  
    - Message: Generated email text  
    - Authentication: Gmail OAuth2 account configured in n8n.  
    - Connect from "Generate the email text".

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Settings: Set your registered HubSpot owner email and Google Sheet link copied from the provided example sheet.                 | https://docs.google.com/spreadsheets/d/1y17jIU6JnNPcmazWf2GsmRpdjBBMnkN41tRJnAO5KrQ/edit?usp=sharing        |
| HubSpot OAuth2 API credential setup instructions with required scopes for contact access.                                        | https://docs.n8n.io/integrations/builtin/credentials/hubspot/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal#required-scopes-for-hubspot-trigger-node |
| LinkedIn Profile Search API requires a RapidAPI key; set up HTTP Header Auth with your key.                                      | https://rapidapi.com/                                                                                        |
| Google Sheets OAuth2 credential is required for reading and updating contact data in sheets.                                      | n8n Google Sheets OAuth2 credential setup.                                                                  |
| For testing, filter a small subset of clients to reduce API calls and speed up debugging.                                         | Sticky Note3 in workflow.                                                                                   |
| The workflow handles HubSpot API pagination to get all contacts, limited to 200 per request.                                     | Sticky Note10 in workflow.                                                                                   |
| Additional tests can be added to detect LinkedIn comments, HubSpot activities, or other updates as needed.                       | Sticky Note5 in workflow.                                                                                   |
| Contact author for help or workflow customization: thomas@pollup.net                                                              | Mailto: thomas@pollup.net                                                                                   |

---

This document fully describes the workflow structure, node-by-node logic, and reproduction instructions to ensure easy understanding, customization, and troubleshooting for both human users and AI automation agents.