Daily Sync of NFL Players from Sleeper API to Airtable for Fantasy Football

https://n8nworkflows.xyz/workflows/daily-sync-of-nfl-players-from-sleeper-api-to-airtable-for-fantasy-football-6602


# Daily Sync of NFL Players from Sleeper API to Airtable for Fantasy Football

---
### 1. Workflow Overview

This workflow automates the daily synchronization of current NFL players’ data from the Sleeper API into an Airtable base for fantasy football management. It targets fantasy football enthusiasts and analysts who require an up-to-date, locally stored snapshot of active NFL players for roster and analytics purposes. The workflow is logically divided into four main blocks:

- **1.1 Input Trigger:** Manual initiation of the sync process.
- **1.2 Data Fetching:** Retrieving the full NFL players dataset from the Sleeper API.
- **1.3 Data Transformation and Filtering:** Converting the fetched data into a manageable array and filtering for active fantasy-relevant players.
- **1.4 Data Upsert:** Creating or updating records in Airtable based on filtered players.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:** Starts the workflow manually to control when the players’ data sync occurs.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’
- **Node Details:**  
  - **Node:** When clicking ‘Execute workflow’  
    - **Type:** Manual Trigger  
    - **Role:** Initiates the workflow on user demand instead of automated scheduling.  
    - **Configuration:** Default manual trigger with no parameters.  
    - **Input/Output:** No input connections; outputs to the HTTP Request node to start data fetching.  
    - **Edge Cases:** None expected; manual activation prevents unintended runs.

#### 1.2 Data Fetching

- **Overview:** Downloads the complete NFL players data from the Sleeper API, which includes every player historically.
- **Nodes Involved:**  
  - Fetch Sleeper NFL Players
- **Node Details:**  
  - **Node:** Fetch Sleeper NFL Players  
    - **Type:** HTTP Request  
    - **Role:** Retrieves raw JSON data containing all NFL players’ information from the Sleeper API endpoint `https://api.sleeper.app/v1/players/nfl`.  
    - **Configuration:** HTTP GET request with default options (no authentication, no query params).  
    - **Key Expressions:** None.  
    - **Input/Output:** Takes trigger input; outputs raw JSON object with player IDs as keys and player objects as values.  
    - **Edge Cases:**  
      - Timeout risk due to large data size (noted in sticky notes).  
      - API unavailability or network errors.  
    - **Sticky Note:** Explains the need to scrape and locally store active players due to API size/timeouts.  
    - **Version Requirements:** n8n HTTP Request node v1 suffices.

#### 1.3 Data Transformation and Filtering

- **Overview:** Converts the large object of players into an array for easier processing, then filters for players who are actively relevant to fantasy football (positions QB, RB, WR, TE with non-null team and name).
- **Nodes Involved:**  
  - Convert Players Object to Array  
  - Filter Active Fantasy Players
- **Node Details:**  
  - **Node:** Convert Players Object to Array  
    - **Type:** Function  
    - **Role:** Transforms the JSON object with player IDs as keys into an array of player objects for iteration.  
    - **Configuration:** JavaScript function iterating over object entries to output an array of `{ json: player }` items.  
    - **Input/Output:** Input is the JSON object from the HTTP Request; output is an array of individual player JSON objects.  
    - **Edge Cases:** Failure if input data is malformed or empty.  
  - **Node:** Filter Active Fantasy Players  
    - **Type:** Function  
    - **Role:** Filters players to only those actively relevant in fantasy football: have a full name, a team, and play one of the key positions QB, RB, WR, or TE.  
    - **Configuration:** JavaScript filter function checking `position`, `full_name`, and `team` fields.  
    - **Input/Output:** Input is array of all players; output is array filtered to active fantasy players.  
    - **Edge Cases:** Missing or null fields could exclude players inadvertently.  
    - **Sticky Note:** Summarizes output fields: player_id, full_name, position, team.

#### 1.4 Data Upsert

- **Overview:** Inserts or updates the filtered active players’ records into an Airtable base, enabling persistent and queryable storage.
- **Nodes Involved:**  
  - Create or update a record
- **Node Details:**  
  - **Node:** Create or update a record  
    - **Type:** Airtable node  
    - **Role:** Performs an upsert operation to synchronize the filtered player list into Airtable.  
    - **Configuration:**  
      - Operation: Upsert  
      - Base and Table selected via credentials (must be configured prior).  
      - Mapping mode set to define fields manually, matching Player_ID as the key.  
    - **Input/Output:** Takes filtered players as input; outputs Airtable operation results (not further connected here).  
    - **Edge Cases:**  
      - Authentication errors if Airtable token is missing or insufficient scope.  
      - Rate limits or API errors from Airtable.  
      - Mismatched field mappings causing failed upserts.  
    - **Sticky Note:** Details required Airtable token scopes for read/write and base schema access. Advises using Player_ID as key.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                       | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                         |
|-----------------------------|----------------------|------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Initiates the workflow manually    |                             | Fetch Sleeper NFL Players    |                                                                                                                                     |
| Fetch Sleeper NFL Players    | HTTP Request         | Fetches full NFL players data       | When clicking ‘Execute workflow’ | Convert Players Object to Array | Explains need to scrape/store players due to API size/timeouts.                                                                     |
| Convert Players Object to Array | Function             | Converts players object to array    | Fetch Sleeper NFL Players    | Filter Active Fantasy Players |                                                                                                                                     |
| Filter Active Fantasy Players | Function             | Filters to active fantasy NFL players | Convert Players Object to Array | Create or update a record     | Outputs player_id, full_name, position, team.                                                                                         |
| Create or update a record    | Airtable             | Upserts active player records       | Filter Active Fantasy Players |                             | Details Airtable Access Token scopes required: data.records:read, data.records:write, schema.bases:read. Use Player_ID as key.       |
| Sticky Note                 | Sticky Note          | Documentation / guidance             |                             |                             | Provides context on workflow’s purpose and data handling.                                                                            |
| Sticky Note1                | Sticky Note          | Airtable credential & API scopes guidance |                             |                             | Advises on setting Airtable token with proper scopes and mapping Player_ID correctly.                                                |
| Sticky Note2                | Sticky Note          | Output fields summary of filtered players |                             |                             | Lists key outputs: player_id, full_name, position, team.                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.  
   - Leave default settings; name it `When clicking ‘Execute workflow’`.

2. **Create HTTP Request Node**  
   - Type: HTTP Request  
   - Name: `Fetch Sleeper NFL Players`  
   - Method: GET  
   - URL: `https://api.sleeper.app/v1/players/nfl`  
   - Leave options default (no authentication needed).  
   - Connect output of `When clicking ‘Execute workflow’` to this node.

3. **Create Function Node to Convert Object to Array**  
   - Type: Function  
   - Name: `Convert Players Object to Array`  
   - Code:  
     ```javascript
     const output = [];
     for (const [key, value] of Object.entries(items[0].json)) {
       output.push({ json: value });
     }
     return output;
     ```  
   - Connect output of `Fetch Sleeper NFL Players` to this node.

4. **Create Function Node to Filter Active Fantasy Players**  
   - Type: Function  
   - Name: `Filter Active Fantasy Players`  
   - Code:  
     ```javascript
     return items.filter(item => {
       const pos = item.json.position;
       const name = item.json.full_name;
       const team = item.json.team;
       return name && team && ['QB','RB','WR','TE'].includes(pos);
     });
     ```  
   - Connect output of `Convert Players Object to Array` to this node.

5. **Create Airtable Node for Upsert**  
   - Type: Airtable  
   - Name: `Create or update a record`  
   - Operation: Upsert  
   - Credentials: Set up Airtable API credentials with appropriate access token:  
     - Scopes required: `data.records:read`, `data.records:write`, `schema.bases:read`.  
   - Select Base and Table matching your target storage for players.  
   - Configure field mapping to match Sleeper’s `player_id` as the unique key for upsert. Map other relevant fields (`full_name`, `position`, `team`).  
   - Connect output of `Filter Active Fantasy Players` to this node.

6. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add notes explaining the large data size from Sleeper API and filtering rationale.  
   - Add instructions on Airtable credential scopes and mapping key fields.  
   - Add a summary note on the output fields for filtered players.

7. **Save and Test Workflow**  
   - Execute manually using the trigger node.  
   - Verify Airtable records create/update as expected.  
   - Monitor for timeouts or errors, especially on the HTTP Request node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow addresses API size/timeout issues by locally syncing active players daily rather than querying Sleeper API repeatedly during fantasy league operations. | Sticky Note on `Fetch Sleeper NFL Players` node |
| Ensure you have an Airtable access token with these scopes: `data.records:read`, `data.records:write`, `schema.bases:read` for successful upsert operations. | Sticky Note1 on Airtable node |
| Use Player_ID as the unique key for mapping and upserting records in Airtable to avoid duplicates and ensure correct updates. | Sticky Note1 and Airtable node configuration |
| Active fantasy players here are defined by positions QB, RB, WR, and TE, and must have both a valid team and full name. | Sticky Note2 on Filter node |
| Sleeper API documentation and endpoints can be found at https://docs.sleeper.app | General reference for expanding or modifying the workflow |

---

**Disclaimer:** The text provided is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.