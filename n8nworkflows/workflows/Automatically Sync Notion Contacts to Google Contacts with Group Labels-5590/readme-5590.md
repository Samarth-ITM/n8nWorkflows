Automatically Sync Notion Contacts to Google Contacts with Group Labels

https://n8nworkflows.xyz/workflows/automatically-sync-notion-contacts-to-google-contacts-with-group-labels-5590


# Automatically Sync Notion Contacts to Google Contacts with Group Labels

### 1. Workflow Overview

This workflow automates the synchronization of contact data from a Notion database to Google Contacts, including the assignment of group labels based on Notion properties. It is designed for users managing contacts in Notion who want to keep their Google Contacts updated automatically, with groups reflecting Notion’s label organization.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Two Notion Trigger nodes detect new or updated contact entries in the Notion database.
- **1.2 Data Retrieval and Preparation:** Retrieves all contacts from Notion, then maps and formats key contact fields for processing.
- **1.3 Sync Status Check:** Determines if each contact has already been synced to avoid duplicates.
- **1.4 Google Contact Groups Fetch and Label Matching:** Retrieves existing Google Contact groups, parses Notion labels, and matches them to Google groups.
- **1.5 Google Contact Group Creation:** Creates new Google Contact groups if matching groups don’t exist.
- **1.6 Contact Addition to Google Contacts:** Adds contacts to Google Contacts, associating them with the correct groups.
- **1.7 Notion Update Post-Sync:** Marks contacts in Notion as synced after successful addition.

Sticky notes provide detailed guidance at various points to clarify operations and configuration.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Detects changes in the Notion contact database to trigger synchronization workflows on new or updated contacts.

**Nodes Involved:**  
- Trigger on New Notion Contact  
- Trigger on Updated Notion Contact

**Node Details:**  

- **Trigger on New Notion Contact**  
  - *Type:* Notion Trigger  
  - *Role:* Fires when a new contact is added in Notion.  
  - *Configuration:* Polls the specified Notion database weekly. Requires setting the Notion database ID.  
  - *Connections:* Output triggers "Get All Contacts from Notion".  
  - *Edge Cases:* Polling intervals may cause delay; missing or incorrect database ID causes failure.  
  - *Sticky Note:* Explains trigger purpose and setup instructions.

- **Trigger on Updated Notion Contact**  
  - *Type:* Notion Trigger  
  - *Role:* Fires when a contact is updated in the Notion database.  
  - *Configuration:* Polls weekly for page updates in the specified Notion database ID.  
  - *Connections:* Output triggers "Get All Contacts from Notion".  
  - *Edge Cases:* Same as above, with potential missed updates if polling is infrequent.  
  - *Sticky Note:* Same as above.

#### 1.2 Data Retrieval and Preparation

**Overview:**  
Retrieves all contacts from Notion and maps relevant fields (name, phone, labels, sync status) into standardized variables for downstream processing.

**Nodes Involved:**  
- Get All Contacts from Notion  
- Map Notion Contact Fields  
- Loop Over Contacts

**Node Details:**  

- **Get All Contacts from Notion**  
  - *Type:* Notion Node  
  - *Role:* Fetches all pages (contacts) from the configured Notion database.  
  - *Configuration:* Returns all entries without pagination limit. Requires the Notion database ID.  
  - *Connections:* Outputs to "Map Notion Contact Fields".  
  - *Edge Cases:* Failure if database ID incorrect or Notion API limits exceeded.  
  - *Sticky Note:* Part of initial data fetching block.

- **Map Notion Contact Fields**  
  - *Type:* Set Node  
  - *Role:* Maps Notion properties to standardized fields: firstName, phone, label(s), and sync status ("Already Added").  
  - *Configuration:* Assigns from Notion JSON fields: `name`, `property_phone`, `property_buy` for labels, and `property_added_to_contacts`.  
  - *Connections:* Outputs to "Loop Over Contacts".  
  - *Edge Cases:* Missing or malformed Notion properties may cause empty or incorrect mappings.  
  - *Sticky Note:* Explains field mapping and customization options.

- **Loop Over Contacts**  
  - *Type:* SplitInBatches  
  - *Role:* Processes contacts one by one or in batches for sequential handling.  
  - *Configuration:* Default batch size, no reset on completion.  
  - *Connections:* Outputs to "Check if Contact Already Synced" (main branch 1) and another branch (main branch 2) for unsynced contacts.  
  - *Edge Cases:* Large datasets require batch tuning to avoid timeouts.

#### 1.3 Sync Status Check

**Overview:**  
Determines if each contact has already been synced to Google Contacts to prevent duplication.

**Nodes Involved:**  
- Check if Contact Already Synced

**Node Details:**  

- **Check if Contact Already Synced**  
  - *Type:* If Node  
  - *Role:* Checks the boolean "Already Added" field mapped from Notion to decide if syncing is needed.  
  - *Configuration:* Condition: `Already Added` is true (boolean).  
  - *Connections:*  
    - If true: loops back to "Loop Over Contacts" to skip syncing.  
    - If false: proceeds to "Fetch Google Contact Groups".  
  - *Edge Cases:* Incorrect or missing "Already Added" field can cause false positives/negatives.

- *Sticky Note:* Describes the logic of checking sync status.

#### 1.4 Google Contact Groups Fetch and Label Matching

**Overview:**  
Retrieves existing contact groups from Google Contacts and matches them against labels from Notion contacts to identify group resource names.

**Nodes Involved:**  
- Fetch Google Contact Groups  
- Find Match Labels

**Node Details:**  

- **Fetch Google Contact Groups**  
  - *Type:* HTTP Request  
  - *Role:* Calls Google People API endpoint to list all contact groups.  
  - *Configuration:* Uses OAuth2 credentials for Google Contacts API, queries up to 1000 groups.  
  - *Connections:* Outputs to "Find Match Labels".  
  - *Edge Cases:* API rate limits, authentication errors, or empty groups list.  
  - *Sticky Note:* Guides on group existence checks.

- **Find Match Labels**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Parses the label(s) from Notion contact data and matches them against fetched Google groups to extract the resourceName.  
  - *Configuration:*  
    - Parses stringified arrays or single string labels.  
    - Matches case-insensitively with Google groups of type "USER_CONTACT_GROUP".  
    - Returns an array of matched labels with corresponding resourceNames.  
  - *Connections:* Outputs to "Check Google Contact Group Exists".  
  - *Edge Cases:* Label parsing errors, unmatched labels, empty label arrays.  
  - *Sticky Note:* Explains label parsing and matching logic.

#### 1.5 Google Contact Group Creation

**Overview:**  
Checks if the group label exists in Google Contacts; if not, creates a new group with that label.

**Nodes Involved:**  
- Check Google Contact Group Exists  
- Create New Google Contact Group

**Node Details:**  

- **Check Google Contact Group Exists**  
  - *Type:* If Node  
  - *Role:* Tests if the matched label resourceName is empty (null or empty array) indicating no existing group.  
  - *Configuration:* Condition: resourceName array empty.  
  - *Connections:*  
    - If true: triggers "Create New Google Contact Group".  
    - If false: proceeds to "Add Contact to Google Contacts".  
  - *Edge Cases:* False negatives if resourceName is malformed or missing.  
  - *Sticky Note:* Describes group existence verification logic.

- **Create New Google Contact Group**  
  - *Type:* HTTP Request  
  - *Role:* Creates a new contact group in Google Contacts using the People API.  
  - *Configuration:*  
    - POST to `https://people.googleapis.com/v1/contactGroups`  
    - JSON body includes the new group name from label.  
    - Uses Google OAuth2 credentials.  
  - *Connections:* Output loops back to "Loop Over Contacts" to retry processing with updated group data.  
  - *Edge Cases:* API quota, naming conflicts, malformed group names.  
  - *Sticky Note:* Explains group creation process.

#### 1.6 Contact Addition to Google Contacts

**Overview:**  
Adds the contact to Google Contacts, associating it with the matched or newly created group labels and including phone number details.

**Nodes Involved:**  
- Add Contact to Google Contacts

**Node Details:**  

- **Add Contact to Google Contacts**  
  - *Type:* Google Contacts Node  
  - *Role:* Creates or updates a contact in Google Contacts with given name, phone, and group label(s).  
  - *Configuration:*  
    - Given name from Notion `firstName` field.  
    - Phone number assigned as home type.  
    - Group field set using Google group’s resourceName array.  
  - *Connections:* Outputs to "Mark Contact as Synced in Notion".  
  - *Edge Cases:* Authentication failures, invalid phone formats, API limits.  
  - *Sticky Note:* Provides guidance on adding contacts with group labels.

#### 1.7 Notion Update Post-Sync

**Overview:**  
Updates the corresponding Notion page to mark the contact as synced, preventing duplicate sync attempts.

**Nodes Involved:**  
- Mark Contact as Synced in Notion

**Node Details:**  

- **Mark Contact as Synced in Notion**  
  - *Type:* Notion Node  
  - *Role:* Updates the “Added to Contacts” checkbox in Notion to true for the synced contact.  
  - *Configuration:*  
    - Uses the page URL from the original Notion contact data to update the correct page.  
    - Sets checkbox property "Added to Contacts" to true.  
  - *Connections:* Outputs back to "Loop Over Contacts" for processing next contact.  
  - *Edge Cases:* Update failures if page URL is invalid or API limits reached.  
  - *Sticky Note:* Explains the purpose of marking contacts as synced.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                         | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                                               |
|-------------------------------|---------------------|---------------------------------------|----------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Trigger on New Notion Contact  | Notion Trigger      | Triggers on new contact addition      | None                             | Get All Contacts from Notion           | Triggers the workflow when a new contact is added or updated in Notion.                                                   |
| Trigger on Updated Notion Contact | Notion Trigger      | Triggers on contact update             | None                             | Get All Contacts from Notion           | Same as above                                                                                                             |
| Get All Contacts from Notion   | Notion              | Retrieves all contacts from Notion    | Trigger on New/Updated Notion    | Map Notion Contact Fields              | Fetches all contacts from configured Notion database.                                                                     |
| Map Notion Contact Fields      | Set                 | Maps Notion fields to variables       | Get All Contacts from Notion     | Loop Over Contacts                     | Maps Notion fields (name, phone, labels, added status) for processing.                                                    |
| Loop Over Contacts             | SplitInBatches      | Processes contacts in batches         | Map Notion Contact Fields        | Check if Contact Already Synced, others | Processes contacts one at a time or in batches.                                                                            |
| Check if Contact Already Synced| If                  | Checks if contact already synced      | Loop Over Contacts               | Loop Over Contacts (if synced), Fetch Google Contact Groups (if not) | Checks the 'Already Added' field to avoid duplicates.                                                                     |
| Fetch Google Contact Groups    | HTTP Request        | Retrieves Google Contact groups       | Check if Contact Already Synced  | Find Match Labels                     | Retrieves up to 1000 contact groups from Google Contacts API.                                                              |
| Find Match Labels              | Code                | Matches Notion labels to Google groups| Fetch Google Contact Groups      | Check Google Contact Group Exists     | Parses labels and matches to Google Contact groups.                                                                        |
| Check Google Contact Group Exists | If                  | Checks existence of group label       | Find Match Labels                | Create New Google Contact Group (if none), Add Contact to Google Contacts (if exists) | Verifies if group exists, else triggers creation.                                                                           |
| Create New Google Contact Group| HTTP Request        | Creates new Google Contact group      | Check Google Contact Group Exists| Loop Over Contacts                     | Creates group if not found.                                                                                                |
| Add Contact to Google Contacts | Google Contacts     | Adds contact with group labels        | Check Google Contact Group Exists| Mark Contact as Synced in Notion      | Adds contact to Google Contacts with name, phone, and group resourceName.                                                 |
| Mark Contact as Synced in Notion| Notion              | Updates Notion to mark contact synced | Add Contact to Google Contacts   | Loop Over Contacts                     | Marks contact as synced in Notion to prevent duplicate syncs.                                                              |
| Workflow Overview             | Sticky Note         | Explanation and setup instructions    | None                             | None                                 | Detailed workflow description and setup instructions.                                                                     |
| Trigger Explanation           | Sticky Note         | Explains trigger nodes                 | None                             | None                                 | Describes the trigger purpose.                                                                                            |
| Field Mapping Guide           | Sticky Note         | Explains field mapping                 | None                             | None                                 | Guides mapping Notion fields to workflow variables.                                                                        |
| Already Synced Check Guide    | Sticky Note         | Explains sync status check             | None                             | None                                 | Describes how the 'Already Added' field is used.                                                                           |
| Label Matching Guide          | Sticky Note         | Explains label matching                | None                             | None                                 | Details label parsing and matching to Google groups.                                                                       |
| Group Existence Check Guide   | Sticky Note         | Explains group existence check        | None                             | None                                 | Describes verification of group existence in Google Contacts.                                                             |
| Create Group Guide            | Sticky Note         | Explains group creation                 | None                             | None                                 | Details creation of new Google Contact groups.                                                                              |
| Add Contact Guide             | Sticky Note         | Explains contact addition               | None                             | None                                 | Describes adding contacts to Google Contacts with group labels.                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Trigger for New Contacts**  
   - Add a Notion Trigger node.  
   - Set polling to run weekly (or preferred interval).  
   - Configure with your Notion credentials.  
   - Set the database ID to your contacts database.  

2. **Create Notion Trigger for Updated Contacts**  
   - Add another Notion Trigger node.  
   - Set event to "pagedUpdatedInDatabase".  
   - Same polling and database ID as above.  

3. **Create "Get All Contacts from Notion" Node**  
   - Add a Notion node.  
   - Set resource to "databasePage", operation "getAll".  
   - Set database ID to your contacts database.  
   - Enable "Return All" to true.  
   - Connect outputs of both triggers to this node.  

4. **Create "Map Notion Contact Fields" Node (Set Node)**  
   - Add a Set node.  
   - Map fields from Notion JSON to variables:  
     - `firstName` = `{{$json["name"]}}`  
     - `phone` = `{{$json["property_phone"]}}`  
     - `label` = `{{$json["property_buy"]}}`  
     - `Already Added` = `{{$json["property_added_to_contacts"]}}`  
   - Connect output of "Get All Contacts from Notion" to this node.  

5. **Create "Loop Over Contacts" Node**  
   - Add SplitInBatches node.  
   - Default batch size (adjust if needed).  
   - Connect from "Map Notion Contact Fields".  

6. **Create "Check if Contact Already Synced" Node (If)**  
   - Add If node.  
   - Condition: `{{$json["Already Added"].toBoolean()}}` is true.  
   - Connect main output of "Loop Over Contacts" to this node.  
   - If true: Connect back to "Loop Over Contacts" to skip syncing.  
   - If false: Connect to "Fetch Google Contact Groups".  

7. **Create "Fetch Google Contact Groups" Node (HTTP Request)**  
   - Add HTTP Request node.  
   - Method: GET  
   - URL: `https://people.googleapis.com/v1/contactGroups`  
   - Query parameter: `pageSize=1000`  
   - Authentication: Google Contacts OAuth2 credentials.  
   - Connect from "Check if Contact Already Synced" (false branch).  

8. **Create "Find Match Labels" Node (Code)**  
   - Add Code node.  
   - Paste JavaScript code that:  
     - Parses `label` field from Notion data (handles strings and arrays).  
     - Matches labels against fetched Google groups by name and type USER_CONTACT_GROUP.  
     - Returns array with name and resourceName array or null.  
   - Connect from "Fetch Google Contact Groups".  

9. **Create "Check Google Contact Group Exists" Node (If)**  
   - Add If node.  
   - Condition: Checks if `resourceName` is empty (null or empty array).  
   - Connect from "Find Match Labels".  
   - If true: Connect to "Create New Google Contact Group".  
   - If false: Connect to "Add Contact to Google Contacts".  

10. **Create "Create New Google Contact Group" Node (HTTP Request)**  
    - Add HTTP Request node.  
    - Method: POST  
    - URL: `https://people.googleapis.com/v1/contactGroups`  
    - Body (JSON): `{ "contactGroup": { "name": "={{ $json.name }}" } }`  
    - Authentication: Google Contacts OAuth2 credentials.  
    - Connect from "Check Google Contact Group Exists" (true branch).  
    - Connect output back to "Loop Over Contacts" (to retry the contact with new group).  

11. **Create "Add Contact to Google Contacts" Node**  
    - Add Google Contacts node.  
    - Operation: Add Contact  
    - Given Name: `={{ $json.firstName }}`  
    - Phone: Set phone type "home", value `={{ $json.phone }}`  
    - Group: Set to `={{ $json.resourceName }}` (from matched labels)  
    - Connect from "Check Google Contact Group Exists" (false branch).  
    - Connect output to "Mark Contact as Synced in Notion".  

12. **Create "Mark Contact as Synced in Notion" Node**  
    - Add Notion node.  
    - Operation: Update page.  
    - Page ID: Use `={{ $('Get All Contacts from Notion').item.json.url }}` to target correct page.  
    - Update property "Added to Contacts" checkbox to true.  
    - Connect from "Add Contact to Google Contacts".  
    - Connect output back to "Loop Over Contacts" to continue processing.  

13. **Add Sticky Notes for Documentation**  
    - Add sticky notes at appropriate positions explaining each block, referring to the detailed content from the original workflow.  
    - Include setup instructions, field mappings, and explanation of label matching and group creation logic.  

14. **Configure Credentials**  
    - Ensure you have valid OAuth2 credentials for Google Contacts and Notion configured in n8n.  
    - Assign credentials to respective nodes requiring authentication.  

15. **Test & Activate Workflow**  
    - Run the workflow manually or wait for triggers.  
    - Monitor logs for errors such as API rate limits, missing fields, or auth failures.  
    - Adjust batch sizes and polling intervals as needed for performance and reliability.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow requires a self-hosted n8n instance as it uses community nodes not available in cloud. | Workflow Overview sticky note.                                                                       |
| For detailed Notion API usage and database ID retrieval, see Notion’s official developer guide.  | https://developers.notion.com/                                                                       |
| OAuth2 credentials must be configured properly for both Notion and Google Contacts nodes.        | Credential prerequisites for the workflow.                                                          |
| Group labels in Google Contacts are matched case-insensitively and must be of type USER_CONTACT_GROUP. | Label Matching Guide sticky note.                                                                    |
| The workflow assumes Notion database fields: name, phone, labels (property_buy), and a checkbox "Added to Contacts". | Field Mapping Guide sticky note.                                                                     |
| Polling intervals (weekly) can be adjusted based on sync frequency needs.                        | Trigger Explanation sticky note.                                                                     |
| Use the “Already Added” checkbox in Notion to prevent duplication and detect synced contacts.   | Already Synced Check Guide sticky note.                                                              |

---

_Disclaimer:_  
The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.