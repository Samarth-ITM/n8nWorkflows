Automatic Trello Board Cleanup by Removing Cards with Specific Labels

https://n8nworkflows.xyz/workflows/automatic-trello-board-cleanup-by-removing-cards-with-specific-labels-7619


# Automatic Trello Board Cleanup by Removing Cards with Specific Labels

### 1. Workflow Overview

This workflow automates the cleanup of a Trello board by deleting cards that have a specific label, namely **"Mark for Deletion"**. It is designed for users who want to maintain tidy Trello boards by automatically removing unwanted cards without manual intervention.

The workflow is structured into the following logical blocks:

- **1.1 Input Trigger**: Manual start of the workflow execution.
- **1.2 Trello Board Retrieval**: Fetch the target Trello board by URL.
- **1.3 List Retrieval**: Get all lists from the board.
- **1.4 Card Retrieval**: For each list, get all cards.
- **1.5 Label Processing**: Split each card’s labels to inspect them individually.
- **1.6 Filtering**: Select only cards labeled “Mark for Deletion.”
- **1.7 Deletion**: Delete the filtered cards from Trello.

Supporting the workflow are sticky notes providing setup instructions and context, including how to connect Trello API credentials and configure the board URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`
- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow execution on user demand.  
  - **Config:** No parameters; simple manual trigger node.  
  - **Inputs:** None  
  - **Outputs:** Triggers next node (`Get Board4`)  
  - **Edge Cases:** None; manual node may require user interaction.  
  - **Sub-workflow:** N/A

#### 2.2 Trello Board Retrieval

- **Overview:** Retrieves the Trello board details using the board URL; serves as the foundation for fetching lists and cards.
- **Nodes Involved:**  
  - `Get Board4`
- **Node Details:**  
  - **Type:** Trello Node (Resource: Board, Operation: Get)  
  - **Role:** Retrieves board info by URL to obtain board ID.  
  - **Config:**  
    - Board ID mode: URL  
    - URL: `https://trello.com/b/DCpuJbnd/administrative-tasks` (user must replace with their own)  
  - **Credentials:** Trello API credentials required (API Key and Token)  
  - **Inputs:** Trigger from manual node  
  - **Outputs:** Board JSON with `id` used by `Get Lists4` and `Get Cards4`  
  - **Edge Cases:** Invalid URL, authentication errors, network issues may cause failure.  
  - **Sub-workflow:** N/A

#### 2.3 List Retrieval

- **Overview:** Fetches all lists from the retrieved Trello board.
- **Nodes Involved:**  
  - `Get Lists4`
- **Node Details:**  
  - **Type:** Trello Node (Resource: List, Operation: Get All)  
  - **Role:** Retrieves all lists on a board using board ID from previous node.  
  - **Config:**  
    - Board ID: expression `={{ $json.id }}` from `Get Board4` output  
  - **Credentials:** Same Trello API credentials as above  
  - **Inputs:** Output of `Get Board4`  
  - **Outputs:** Lists JSON passed to `Get Cards4`  
  - **Edge Cases:** Empty or inaccessible board lists, API rate limiting, auth errors.

#### 2.4 Card Retrieval

- **Overview:** For each list, retrieves all cards.
- **Nodes Involved:**  
  - `Get Cards4`
- **Node Details:**  
  - **Type:** Trello Node (Resource: List, Operation: Get Cards)  
  - **Role:** Retrieves cards for each list ID.  
  - **Config:**  
    - List ID: expression `={{ $json.id }}` from `Get Lists4` output  
  - **Credentials:** Trello API credentials as above  
  - **Inputs:** Output from `Get Lists4`  
  - **Outputs:** Cards JSON passed to `Split Labels`  
  - **Edge Cases:** Empty lists, API limits, connection errors.

#### 2.5 Label Processing

- **Overview:** Splits the labels array of each card into individual items to facilitate filtering by label name.
- **Nodes Involved:**  
  - `Split Labels`
- **Node Details:**  
  - **Type:** Split Out (built-in n8n node)  
  - **Role:** Splits the array field `labels` into separate items for filtering.  
  - **Config:**  
    - Field to split out: `labels`  
  - **Inputs:** Cards array from `Get Cards4`  
  - **Outputs:** Individual label objects passed to filter node  
  - **Edge Cases:** Cards with no labels result in no output; empty labels array.

#### 2.6 Filtering

- **Overview:** Filters labels to find those exactly named “Mark for Deletion,” thereby identifying cards to delete.
- **Nodes Involved:**  
  - `Filter Marked for Delete`
- **Node Details:**  
  - **Type:** Filter (version 2.2)  
  - **Role:** Only passes labels where the label name equals “Mark for Deletion” (case-sensitive).  
  - **Config:**  
    - Condition: `$json.name === "Mark for Deletion"`  
  - **Inputs:** Individual label items from `Split Labels`  
  - **Outputs:** Labels matching the condition, triggering deletion node  
  - **Edge Cases:** Case sensitivity may cause misses; no label match results in no deletion.

#### 2.7 Deletion

- **Overview:** Deletes cards identified with the “Mark for Deletion” label.
- **Nodes Involved:**  
  - `Delete a card`
- **Node Details:**  
  - **Type:** Trello Node (Resource: Card, Operation: Delete)  
  - **Role:** Deletes card by ID.  
  - **Config:**  
    - Card ID: expression `={{ $('Get Cards4').item.json.id }}` — references the original card ID, not label ID  
  - **Credentials:** Trello API credentials as above  
  - **Inputs:** Cards filtered by label in `Filter Marked for Delete`  
  - **Outputs:** Deletion result (success/failure)  
  - **Edge Cases:** Deletion failure if card already deleted, permission issues, API errors.

---

### 3. Summary Table

| Node Name                 | Node Type       | Functional Role                         | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                             |
|---------------------------|-----------------|---------------------------------------|---------------------------|-------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger  | Workflow manual start trigger          | —                         | Get Board4              |                                                                                                       |
| Get Board4                | Trello          | Retrieve board info by URL              | When clicking ‘Execute workflow’ | Get Lists4              |                                                                                                       |
| Get Lists4                | Trello          | Get all lists from board                | Get Board4                 | Get Cards4              |                                                                                                       |
| Get Cards4                | Trello          | Get all cards from each list            | Get Lists4                 | Split Labels            |                                                                                                       |
| Split Labels              | Split Out       | Split labels array into individual labels | Get Cards4                 | Filter Marked for Delete |                                                                                                       |
| Filter Marked for Delete  | Filter          | Filter cards with label “Mark for Deletion” | Split Labels               | Delete a card           |                                                                                                       |
| Delete a card             | Trello          | Delete cards identified for deletion    | Filter Marked for Delete   | —                       |                                                                                                       |
| Sticky Note5              | Sticky Note     | Setup instructions for Trello API       | —                         | —                       | ## ⚙️ Setup Instructions... (see setup steps for Trello credentials and board URL)                     |
| Sticky Note56             | Sticky Note     | Workflow purpose and overview            | —                         | —                       | # 🗑️ Auto Trello Cleanup with Label and Delete Card Node... (explains workflow goal and usage)         |
| Sticky Note58             | Sticky Note     | Trello API connection instructions      | —                         | —                       | ### 1️⃣ Connect Trello (Developer API)... (API key and token instructions)                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters  
   - This node starts the workflow manually.

2. **Create Trello Node to Get Board**  
   - Name: `Get Board4`  
   - Type: Trello  
   - Resource: Board  
   - Operation: Get  
   - ID Mode: URL  
   - URL: Paste your Trello board URL (e.g., `https://trello.com/b/DCpuJbnd/administrative-tasks`)  
   - Credentials: Configure Trello API credentials (API Key and Token)  
   - Connect output of Manual Trigger node to this node.

3. **Create Trello Node to Get Lists**  
   - Name: `Get Lists4`  
   - Type: Trello  
   - Resource: List  
   - Operation: Get All  
   - Board ID: Use expression `={{ $json.id }}` to get board ID from `Get Board4` node output  
   - Credentials: Same Trello API credentials  
   - Connect output of `Get Board4` to this node.

4. **Create Trello Node to Get Cards**  
   - Name: `Get Cards4`  
   - Type: Trello  
   - Resource: List  
   - Operation: Get Cards  
   - List ID: Use expression `={{ $json.id }}` to get list ID from `Get Lists4` output  
   - Credentials: Same Trello API credentials  
   - Connect output of `Get Lists4` to this node.

5. **Create Split Out Node**  
   - Name: `Split Labels`  
   - Type: Split Out  
   - Field to Split Out: `labels` (the label array from cards)  
   - Connect output of `Get Cards4` to this node.

6. **Create Filter Node**  
   - Name: `Filter Marked for Delete`  
   - Type: Filter (Version 2.2)  
   - Conditions:  
     - Left Value: Expression `{{$json.name}}` (label name)  
     - Operator: Equals  
     - Right Value: `Mark for Deletion` (case-sensitive exact match)  
   - Connect output of `Split Labels` to this node.

7. **Create Trello Node to Delete Card**  
   - Name: `Delete a card`  
   - Type: Trello  
   - Resource: Card  
   - Operation: Delete  
   - Card ID: Use expression `={{ $('Get Cards4').item.json.id }}` to reference the card ID corresponding to the matched label.  
   - Credentials: Same Trello API credentials  
   - Connect output of `Filter Marked for Delete` to this node.

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add notes with setup instructions for Trello API credentials and workflow explanation as per the original sticky notes content.

**Credential Setup:**  
- In n8n, go to Credentials → New → Trello API  
- Paste your Trello API Key and Token (available at https://trello.com/app-key)  
- Save and select these credentials in all Trello nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| To connect Trello API: obtain API Key at https://trello.com/app-key and generate a token on the same page.                      | Trello API Setup instructions from Sticky Note5 and Sticky Note58                                       |
| Workflow automatically deletes cards labeled “Mark for Deletion” to keep boards tidy without manual intervention.                | Workflow Description in Sticky Note56                                                                    |
| Contact Robert Breen for support or questions: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Contact info from Sticky Note5                                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.