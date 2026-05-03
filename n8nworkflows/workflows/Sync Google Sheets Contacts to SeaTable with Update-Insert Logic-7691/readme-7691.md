Sync Google Sheets Contacts to SeaTable with Update/Insert Logic

https://n8nworkflows.xyz/workflows/sync-google-sheets-contacts-to-seatable-with-update-insert-logic-7691


# Sync Google Sheets Contacts to SeaTable with Update/Insert Logic

### 1. Workflow Overview

This workflow automates the synchronization of contacts from a Google Sheets spreadsheet to a SeaTable base, ensuring each contact is either updated if it exists or inserted if not. It is designed for users who maintain their contact lists in Google Sheets but want to keep an up-to-date record in SeaTable for further data management or collaboration.

The workflow is logically divided into the following blocks:

- **1.1 Initialization and Settings:** Load configuration parameters such as Google Sheet ID and SeaTable credentials.
- **1.2 Fetch Contacts from Google Sheets:** Retrieve the list of contacts from the configured Google Sheet.
- **1.3 Process Each Contact:** Loop over each contact and check if it exists in SeaTable.
- **1.4 Conditional Logic for Upsert:** For each contact, decide to update an existing row or create a new one in SeaTable.
- **1.5 SeaTable API Setup (Optional):** Prepare or create the SeaTable table schema via HTTP API calls.
- **1.6 Documentation and User Guidance:** Sticky notes providing instructions, links, and explanations.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Settings

**Overview:**  
This block sets up essential parameters such as the Google Sheet ID and SeaTable API credentials needed for subsequent API calls and integrations.

**Nodes Involved:**  
- `When clicking ‘Execute workflow’` (Manual Trigger)  
- `settings` (Set)  
- `Edit Fields` (Set)  
- `HTTP Request1` (HTTP Request)  
- `Edit Fields2` (Set)  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow  
  - Config: No parameters; triggers workflow run on user action  
  - Inputs: None  
  - Outputs: Connects to `settings` node  
  - Edge cases: None  

- **settings**  
  - Type: Set  
  - Role: Define and store Google Sheet ID  
  - Config: Assigns a string variable `googlesheetid` (placeholder `<put your Google Sheet ID here>`)  
  - Inputs: From Manual Trigger  
  - Outputs: Connects to `contacts` node  
  - Edge cases: If user does not replace the placeholder, subsequent nodes will fail due to invalid Sheet ID  

- **Edit Fields**  
  - Type: Set  
  - Role: Store SeaTable API parameters (`server`, `base_uuid`, `access_token`)  
  - Config: Three string fields with placeholders for server domain, base UUID, and access token  
  - Inputs: From `HTTP Request1` node  
  - Outputs: Connects to `HTTP Request` node  
  - Edge cases: Invalid or missing tokens will cause API authorization failures  

- **HTTP Request1**  
  - Type: HTTP Request  
  - Role: Retrieve server info from SeaTable, likely for dynamic info or validation  
  - Config: GET request to `https://cloud.seatable.io/server-info` with Accept header set to `application/json`  
  - Inputs: From `Edit Fields2` (actually no inputs; connection from `HTTP Request1` to `Edit Fields2`)  
  - Outputs: Connects to `Edit Fields2`  
  - Edge cases: Network failures, unauthorized requests, or invalid endpoint response  

- **Edit Fields2**  
  - Type: Set  
  - Role: Placeholder for further parameter adjustments or to store server info response  
  - Config: No explicit parameters defined, likely used as pass-through or for future use  
  - Inputs: From `HTTP Request1`  
  - Outputs: Connects to `Edit Fields`  
  - Edge cases: None  

---

#### 2.2 Fetch Contacts from Google Sheets

**Overview:**  
This block retrieves the list of contacts from the specified Google Sheets document to be processed.

**Nodes Involved:**  
- `contacts` (Google Sheets)  

**Node Details:**  

- **contacts**  
  - Type: Google Sheets  
  - Role: Fetch data from a specific sheet named "contacts" within the Google Sheets document identified by `googlesheetid`  
  - Config:  
    - Document ID is set dynamically from the variable `googlesheetid`  
    - Sheet name fixed to the sheet with ID `1877973285` (interpreted as "contacts")  
    - No additional options configured, default read behavior  
  - Inputs: From `settings` node  
  - Outputs: Connects to `Loop Over Items` node  
  - Edge cases:  
    - Invalid or inaccessible Google Sheet ID leads to authorization or not found errors  
    - Empty or malformed sheet data may cause processing issues  

---

#### 2.3 Process Each Contact

**Overview:**  
The workflow iterates through each contact fetched from Google Sheets to process them individually.

**Nodes Involved:**  
- `Loop Over Items` (SplitInBatches)  
- `Replace Me` (NoOp)  
- `seatablelookup` (SeaTable Search)  
- `If` (Conditional)  
- `Create a row` (SeaTable Create)  
- `Update a row` (SeaTable Update)  

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterate over each contact record one-by-one for controlled processing  
  - Config: Default batch size (split one item at a time)  
  - Inputs: From `contacts` node  
  - Outputs: Two outputs: one to `Replace Me` (no operation node), another to `seatablelookup`  
  - Edge cases: Large datasets may require batch size tuning for performance  

- **Replace Me**  
  - Type: NoOp  
  - Role: Placeholder node with no operation; potentially for future extension or debugging  
  - Config: Execute once, no parameters  
  - Inputs: From `Loop Over Items` (first output)  
  - Outputs: None  
  - Edge cases: None  

- **seatablelookup**  
  - Type: SeaTable  
  - Role: Search for existing contact in SeaTable by email to determine if update or insert is needed  
  - Config:  
    - Operation: Search  
    - Table name: "Table1"  
    - Search term: Dynamic, current contact's `email` field  
    - Search column: "email"  
  - Credentials: SeaTable API credentials configured externally  
  - Inputs: From `Loop Over Items` (second output)  
  - Outputs: Connects to `If` node  
  - Edge cases:  
    - API failures or quota limits  
    - Missing or malformed emails may lead to false negatives  

- **If**  
  - Type: Conditional  
  - Role: Branch the flow based on whether the SeaTable lookup returned an empty result (`isEmpty` true)  
  - Config:  
    - Condition: Check if `$json.isEmpty()` is true (indicates no existing record found)  
  - Inputs: From `seatablelookup`  
  - Outputs: Two branches  
    - True branch: Connects to `Create a row` (insert new record)  
    - False branch: Connects to `Update a row` (update existing record)  
  - Edge cases: Logic depends on correct detection of empty results  

- **Create a row**  
  - Type: SeaTable  
  - Role: Insert new contact record into SeaTable table "Table1"  
  - Config:  
    - Columns set dynamically from current contact fields: `email`, `firstname`, `lastname`, `company`  
  - Inputs: From `If` node (true branch)  
  - Outputs: Connects back to `Loop Over Items` (to continue processing)  
  - Edge cases: API errors, data validation failures, duplicate insertion risks if conditional logic fails  

- **Update a row**  
  - Type: SeaTable  
  - Role: Update an existing contact record identified by the `_id` from SeaTable search  
  - Config:  
    - Row ID set dynamically from `$json._id` of found record  
    - Columns updated with contact fields from Google Sheets  
  - Inputs: From `If` node (false branch)  
  - Outputs: Connects back to `Loop Over Items`  
  - Edge cases:  
    - Row ID missing or stale leads to update failure  
    - API errors and conflicts if multiple updates occur concurrently  

---

#### 2.4 SeaTable API Setup (Optional)

**Overview:**  
This optional block demonstrates how to create the SeaTable table schema via API call, useful for initial setup or automation of SeaTable environment.

**Nodes Involved:**  
- `HTTP Request`  

**Node Details:**  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Create a table named "contact" with specified columns (`email`, `firstname`, `lastname`, `company`) in SeaTable using API  
  - Config:  
    - Method: POST  
    - URL dynamically built from server and base UUID variables  
    - JSON body defines table and columns structure  
    - Authorization header uses Bearer token from variables  
  - Inputs: From `Edit Fields` node  
  - Outputs: None  
  - Edge cases:  
    - Authorization errors due to invalid token  
    - HTTP failures or API limits  
    - Table already existing causes errors  

---

#### 2.5 Documentation and User Guidance

**Overview:**  
Sticky Notes provide instructions, documentation, and useful links to help users understand and configure the workflow.

**Nodes Involved:**  
- `Sticky Note1`  
- `Sticky Note2`  
- `Sticky Note6`  
- `Sticky Note7`  
- `Sticky Note8`  

**Node Details:**  

- **Sticky Note1**  
  - Content: Detailed instructions and links for SeaTable API usage, token generation, and base UUID retrieval  
  - Role: Reference for SeaTable API setup and credentials  

- **Sticky Note2**  
  - Content: "# Get Server Info"  
  - Role: Label for `HTTP Request1` node  

- **Sticky Note6**  
  - Content: "Set the Google Sheet ID you want to use"  
  - Role: Reminder for user to update `settings` node  

- **Sticky Note7**  
  - Content: Extensive user guide detailing workflow purpose, usage steps, Google Sheet template link, requirements, and support contacts  
  - Role: Comprehensive overview and instructions for end users  

- **Sticky Note8**  
  - Content: Link to [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing)  
  - Role: Quick access to template for contacts  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                     | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                              |
|-------------------------|---------------------|----------------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Entry point to start workflow                       | None                             | settings                        |                                                                                                        |
| settings                | Set                 | Configure Google Sheet ID                           | When clicking ‘Execute workflow’ | contacts                       | Set the Google Sheet ID you want to use                                                                |
| contacts                | Google Sheets       | Fetch contacts from Google Sheets                   | settings                        | Loop Over Items                | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |
| Loop Over Items         | SplitInBatches      | Iterate over each contact record                     | contacts                       | Replace Me; seatablelookup     |                                                                                                        |
| Replace Me              | NoOp                | Placeholder, no operation                            | Loop Over Items                | None                          |                                                                                                        |
| seatablelookup          | SeaTable            | Search for contact by email in SeaTable             | Loop Over Items                | If                            |                                                                                                        |
| If                      | If                  | Decide to insert or update based on search result   | seatablelookup                 | Create a row (true); Update a row (false) |                                                                                                        |
| Create a row            | SeaTable            | Insert new contact record                            | If (true branch)               | Loop Over Items               |                                                                                                        |
| Update a row            | SeaTable            | Update existing contact record                       | If (false branch)              | Loop Over Items               |                                                                                                        |
| Edit Fields             | Set                 | Store SeaTable API credentials and parameters       | Edit Fields2                   | HTTP Request                  | # Create Table via API  https://api.seatable.com/reference/createtable ...                              |
| HTTP Request            | HTTP Request        | Create SeaTable table via API                        | Edit Fields                   | None                         | # Create Table via API  https://api.seatable.com/reference/createtable ...                              |
| HTTP Request1           | HTTP Request        | Get SeaTable server info                             | None                          | Edit Fields2                  | # Get Server Info                                                                                       |
| Edit Fields2            | Set                 | Placeholder for server info or parameters            | HTTP Request1                 | Edit Fields                  |                                                                                                        |
| Sticky Note1            | Sticky Note         | Instructions for SeaTable API and tokens             | None                          | None                         | # Create Table via API  https://api.seatable.com/reference/createtable ...                              |
| Sticky Note2            | Sticky Note         | Label for Get Server Info node                        | None                          | None                         | # Get Server Info                                                                                       |
| Sticky Note6            | Sticky Note         | Reminder to set Google Sheet ID                       | None                          | None                         | Set the Google Sheet ID you want to use                                                                |
| Sticky Note7            | Sticky Note         | Comprehensive workflow explanation and instructions | None                          | None                         | # Keep your Google Sheets contacts in sync with SeaTable ...                                           |
| Sticky Note8            | Sticky Note         | Google Sheet template link                            | None                          | None                         | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named `When clicking ‘Execute workflow’`. No configuration needed.

2. **Create Set Node for Google Sheet ID**  
   - Add a Set node named `settings`.  
   - Create a string field named `googlesheetid`.  
   - Set the value to your Google Sheet document ID (string between `/d/` and `/edit` in the URL).  
   - Connect `When clicking ‘Execute workflow’` to `settings`.

3. **Create Google Sheets Node to Fetch Contacts**  
   - Add a Google Sheets node named `contacts`.  
   - Select operation to read rows from a sheet.  
   - Set document ID to the expression: `={{ $json.googlesheetid }}` to get from `settings` node.  
   - Set sheet name or ID to the sheet containing contacts (ID `1877973285` or your sheet name).  
   - Connect `settings` to `contacts`.  
   - Configure Google Sheets OAuth2 credentials.

4. **Create SplitInBatches Node for Iteration**  
   - Add a SplitInBatches node named `Loop Over Items`.  
   - Default batch size (1 item per batch).  
   - Connect `contacts` to `Loop Over Items`.

5. **Add NoOp Node as Placeholder**  
   - Add a NoOp node named `Replace Me`.  
   - Set to execute once (optional for debugging).  
   - Connect first output of `Loop Over Items` to `Replace Me`.

6. **Add SeaTable Search Node**  
   - Add a SeaTable node named `seatablelookup`.  
   - Set operation to `search`.  
   - Table name: `Table1`.  
   - Search term: Use expression to get current item’s email: `={{ $json.email }}`.  
   - Search column: `email`.  
   - Connect second output of `Loop Over Items` to `seatablelookup`.  
   - Configure SeaTable API credentials.

7. **Add If Node for Conditional Logic**  
   - Add an If node named `If`.  
   - Condition: Check if `$json.isEmpty()` is true (indicating no existing record found).  
   - Connect output of `seatablelookup` to `If`.

8. **Add SeaTable Create Node**  
   - Add a SeaTable node named `Create a row`.  
   - Operation: `create`.  
   - Table name: `Table1`.  
   - Map columns:  
     - `email` → `={{ $('contacts').item.json.email }}`  
     - `firstname` → `={{ $('contacts').item.json.firstname }}`  
     - `lastname` → `={{ $('contacts').item.json.lastname }}`  
     - `company` → `={{ $('contacts').item.json.company }}`  
   - Connect `If` node’s true output to `Create a row`.  
   - Connect output of `Create a row` back to `Loop Over Items` to continue processing.

9. **Add SeaTable Update Node**  
   - Add a SeaTable node named `Update a row`.  
   - Operation: `update`.  
   - Table name: `Table1`.  
   - Row ID: `={{ $json._id }}` (from `seatablelookup` result).  
   - Map columns same as in create node.  
   - Connect `If` node’s false output to `Update a row`.  
   - Connect output of `Update a row` back to `Loop Over Items`.

10. **(Optional) Add Set Node for SeaTable API Credentials**  
    - Add a Set node named `Edit Fields`.  
    - Define string fields:  
      - `server` (e.g., `cloud.seatable.io`)  
      - `base_uuid` (your SeaTable base UUID)  
      - `access_token` (your SeaTable access token)  
    - Connect this node appropriately if using the API setup flow.

11. **(Optional) Add HTTP Request Node to Create Table**  
    - Add an HTTP Request node named `HTTP Request`.  
    - Method: POST.  
    - URL: `https://{{ $json.server }}/api-gateway/api/v2/dtables/{{ $json.base_uuid }}/tables`.  
    - Headers:  
      - `accept: application/json`  
      - `authorization: Bearer {{ $json.access_token }}`  
    - Body (JSON): Define table name and columns as per workflow.  
    - Connect `Edit Fields` to this node.

12. **(Optional) Add HTTP Request Node to Get Server Info**  
    - Add HTTP Request node named `HTTP Request1`.  
    - URL: `https://cloud.seatable.io/server-info`.  
    - Method: GET.  
    - Header accept: `application/json`.  
    - Connect to a Set node named `Edit Fields2` to process or store results.

13. **Add Sticky Notes**  
    - Use sticky notes for user instructions at relevant points:  
      - Reminder to set Google Sheet ID near `settings` node.  
      - Link to Google Sheet template near `contacts` node.  
      - Detailed instructions and references near SeaTable credential nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Keep your Google Sheets contacts in sync with SeaTable. Update or Insert records in SeaTable as per contact existence. Use the provided Google Sheet template to structure your data. Requires Google credentials and SeaTable cloud access. Tested on n8n version 1.105.2 (Ubuntu). For help, visit the [LinkedIn post](https://www.linkedin.com/posts/n8n-about_n8n-seatable-airtable-activity-7363129613465571332-Scun/) or contact the author on [LinkedIn](https://www.linkedin.com/in/stephaneheckel/) or the [n8n Forum](https://community.n8n.io/). | Sticky Note7                                                                                                   |
| Google Sheet Template for contacts synchronization: https://docs.google.com/spreadsheets/d/1mFKp3wmbV9qp2tpGGsN72zdiC32y8H1nhjdgP885y-U/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                             | Sticky Note8                                                                                                   |
| SeaTable API and credentials setup instructions: How to create tables, get tokens, and find base UUIDs. Reference links: https://api.seatable.com/reference/createtable, https://api.seatable.com/reference/getbasetokenwithapitoken, https://account.seatable.com/bases, https://api.seatable.com/reference/createapitoken, https://account.seatable.com/api, https://cloud.seatable.io/workspace/85603/dtable/datanosco/?tid=0000&vid=0000                                                                                                   | Sticky Note1                                                                                                   |
| Reminder to set the Google Sheet ID in the `settings` node before running the workflow.                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note6                                                                                                   |
| Label for the HTTP Request node retrieving SeaTable server information.                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note2                                                                                                   |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.