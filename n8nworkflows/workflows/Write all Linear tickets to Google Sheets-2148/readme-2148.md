Write all Linear tickets to Google Sheets

https://n8nworkflows.xyz/workflows/write-all-linear-tickets-to-google-sheets-2148


# Write all Linear tickets to Google Sheets

### 1. Workflow Overview

This workflow automates the synchronization of all tickets from specified Linear teams into a Google Sheets spreadsheet. It is designed for teams who want to perform custom analysis or reporting on Linear issues without paying for Linear's Plus Insights features or when the built-in insights do not cover their needs.

The workflow runs daily at a specified time and performs the following logical blocks:

- **1.1 Scheduled Trigger:** Triggers the workflow automatically every day at 09:00 UTC.
- **1.2 Fetch Linear Tickets:** Queries the Linear API via GraphQL to retrieve tickets for a configured team, including pagination support to handle more than 100 tickets.
- **1.3 Pagination Handling:** Checks if more pages of tickets exist and fetches them iteratively until all tickets are retrieved.
- **1.4 Ticket Processing:** Splits the fetched tickets into individual items, applies custom field transformations such as default estimates and label formatting.
- **1.5 Data Flattening:** Flattens nested ticket objects into a simple key-value structure suitable for Google Sheets.
- **1.6 Write to Google Sheets:** Appends or updates the tickets into a specified Google Sheets spreadsheet, matching on the ticket ID to avoid duplicates.

Several sticky notes provide configuration guidance for customizing team names, Google Sheets target, and custom ticket fields.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Initiates the workflow automatically at 09:00 UTC every day.
- **Nodes Involved:** 
  - Every day at 06:00 (Schedule Trigger)
- **Node Details:**
  - **Type:** Schedule Trigger
  - **Configuration:** Set to trigger daily at hour 9 (09:00 UTC).
  - **Inputs:** None
  - **Outputs:** Triggers the GraphQL query node to start fetching tickets.
  - **Edge Cases:** If the n8n instance is offline or suspended at trigger time, the run will be skipped.
  - **Version:** 1

#### 2.2 Fetch Linear Tickets

- **Overview:** Queries Linear’s GraphQL API to fetch the first page (up to 100) of tickets for a specified team.
- **Nodes Involved:** 
  - Get all your team's tickets (GraphQL)
  - Sticky Note1 (Instruction)
- **Node Details:**
  - **Type:** GraphQL
  - **Configuration:**
    - Query fetches fields: id, identifier, url, title, priorityLabel, createdAt, completedAt, state (type & name), cycle number, estimate, and labels.
    - Filter uses team name equality; default team name set to "Adore".
    - Authentication uses HTTP header with Linear API key.
  - **Inputs:** Trigger from Schedule node.
  - **Outputs:** JSON with issues and pageInfo for pagination.
  - **Expressions:** Filter team name is hardcoded but can be customized by editing variables.
  - **Edge Cases:** API key errors, network timeouts, team name typo leading to empty results.
  - **Version:** 1

#### 2.3 Pagination Handling

- **Overview:** Checks if more pages exist via `hasNextPage` flag and uses the `endCursor` to fetch subsequent pages recursively.
- **Nodes Involved:** 
  - if has next page (If)
  - Get end cursor (Set)
  - Get next page (GraphQL)
  - Sticky Note (Instruction near Get next page)
- **Node Details:**
  - **If Node:**
    - Checks if `data.issues.pageInfo.hasNextPage` is true.
    - If true, proceeds to Set node.
  - **Set Node:**
    - Extracts `endCursor` from the API response to pass as `after` variable.
  - **Get next page Node:**
    - Similar GraphQL query as the first page, but includes `after` cursor to fetch next batch.
    - Filter team name must be consistent with initial query.
  - **Inputs/Outputs:**
    - Loops back to `if has next page` to continue pagination until no more pages.
  - **Edge Cases:**
    - Infinite loops if API pagination is inconsistent.
    - API rate limits or failures.
  - **Version:** If node v2, Set node v3.2, GraphQL node v1.

#### 2.4 Ticket Processing

- **Overview:** Breaks the tickets array into individual ticket items and applies custom field modifications such as setting a default estimate if missing and converting label arrays to comma-separated strings.
- **Nodes Involved:** 
  - Split out the tickets (SplitOut)
  - Set custom fields (Set)
  - Sticky Note2 (Instruction)
- **Node Details:**
  - **SplitOut Node:**
    - Splits the array of tickets `data.issues.nodes` into separate items for easier per-ticket processing.
  - **Set Node:**
    - Sets `estimate` field to ticket’s estimate or defaults to 1 if null.
    - Converts labels from nested objects to a CSV string of label names.
  - **Inputs/Outputs:** Receives array from GraphQL nodes; outputs processed individual tickets.
  - **Edge Cases:** Tickets without any labels or estimate fields.
  - **Version:** SplitOut v1, Set v3.2

#### 2.5 Data Flattening

- **Overview:** Flattens nested ticket JSON objects (e.g., state, cycle) into a flat structure with dot notation keys for straightforward mapping in Google Sheets.
- **Nodes Involved:** 
  - Flatten object to have simple fields to filter by (Code)
- **Node Details:**
  - **Type:** Code (JavaScript)
  - **Configuration:** Runs once per ticket item; recursively flattens nested objects.
  - **Inputs:** Individual ticket JSON objects.
  - **Outputs:** Flattened JSON with keys like `state.type` instead of nested objects.
  - **Edge Cases:** Deeply nested or unexpected null values.
  - **Version:** 2

#### 2.6 Write to Google Sheets

- **Overview:** Appends or updates each ticket row in a specified Google Sheets spreadsheet using `id` as a unique key.
- **Nodes Involved:** 
  - Write tickets to Sheets (Google Sheets)
  - Sticky Note4 (Instruction)
- **Node Details:**
  - **Type:** Google Sheets
  - **Configuration:**
    - Operation: appendOrUpdate
    - Mapping mode: autoMapInputData
    - Matching column: id (ticket ID)
    - Spreadsheet and sheet specified via dropdown from connected Google account.
  - **Inputs:** Flattened ticket JSON objects.
  - **Outputs:** Updated spreadsheet rows.
  - **Credentials:** Uses Google Sheets OAuth2 credentials.
  - **Edge Cases:**
    - Google API quota limits.
    - Permissions errors.
    - Mismatches if `id` column is missing in sheet.
  - **Version:** 4.2

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                        | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                      |
|-----------------------------|---------------------|-------------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Every day at 06:00           | Schedule Trigger    | Starts workflow daily at 09:00 UTC  | None                      | Get all your team's tickets  |                                                                                                |
| Get all your team's tickets  | GraphQL             | Fetches first 100 tickets by team   | Every day at 06:00         | Split out the tickets, if has next page | 👇🏽 Set your team name here in the filter. (Our team's name is Adore)                          |
| Sticky Note1                 | Sticky Note         | Instruction for team name setting    | None                      | None                        | 👇🏽 Set your team name here in the filter. (Our team's name is Adore)                          |
| if has next page             | If                  | Checks if more pages exist           | Get all your team's tickets, Get next page | Get end cursor, Split out the tickets |                                                                                                |
| Get end cursor               | Set                 | Extracts pagination cursor           | if has next page           | Get next page               |                                                                                                |
| Get next page                | GraphQL             | Fetches next page of tickets         | Get end cursor             | if has next page, Split out the tickets | 👈🏽 Set your team name here in the filter. (Our team's name is Adore)                          |
| Sticky Note                 | Sticky Note         | Instruction for team name setting    | None                      | None                        | 👈🏽 Set your team name here in the filter. (Our team's name is Adore)                          |
| Split out the tickets        | SplitOut             | Splits tickets array into items      | Get all your team's tickets, Get next page | Set custom fields           |                                                                                                |
| Set custom fields            | Set                 | Sets default estimate and formats labels | Split out the tickets      | Flatten object to have simple fields to filter by | 👇🏽 Adjust any custom fields. Here we set labels and default estimate of 1                      |
| Sticky Note2                 | Sticky Note         | Instruction for custom fields setup  | None                      | None                        | 👇🏽 Adjust any custom fields. Here we set labels and default estimate of 1                      |
| Flatten object to have simple fields to filter by | Code (JavaScript) | Flattens nested ticket JSON        | Set custom fields          | Write tickets to Sheets     |                                                                                                |
| Write tickets to Sheets      | Google Sheets       | Writes/updates tickets in spreadsheet | Flatten object to have simple fields to filter by | None                        | 👆🏽 Update which Google sheet to write to                                                       |
| Sticky Note4                 | Sticky Note         | Instruction for Google Sheets config | None                      | None                        | 👆🏽 Update which Google sheet to write to                                                       |
| Sticky Note3                 | Sticky Note         | Setup instructions overview          | None                      | None                        | ### 👨‍🎤 Setup 1. Add Linear API header key 2. Add Google sheets creds 3. Update which teams... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Name: `Every day at 06:00`
   - Set to trigger daily at hour 9 (09:00 UTC).

2. **Create a GraphQL node:**
   - Name: `Get all your team's tickets`
   - Endpoint: `https://api.linear.app/graphql`
   - Query:
     ```graphql
     query ($filter: IssueFilter) {
       issues(filter: $filter, first: 100) {
         pageInfo {
           hasNextPage
           endCursor
         }
         nodes {
           id
           identifier
           url
           title
           priorityLabel
           createdAt
           completedAt
           state {
             type
             name
           }
           cycle {
             number
           }
           estimate
           labels { nodes { name } }
         }
       }
     }
     ```
   - Variables:
     ```json
     {
       "filter": {
         "team": {
           "name": {
             "eq": "Adore"
           }
         }
       }
     }
     ```
   - Authentication: HTTP Header Auth with your Linear API key.
   - Connect: Output of Schedule Trigger to this node’s input.

3. **Create an If node:**
   - Name: `if has next page`
   - Condition: Check if `{{$json["data"]["issues"]["pageInfo"]["hasNextPage"]}}` equals boolean `true`.
   - Connect the GraphQL node output to this node.

4. **Create a Set node:**
   - Name: `Get end cursor`
   - Set variable `after` to `{{$json["data"]["issues"]["pageInfo"]["endCursor"]}}`.
   - Connect the True output of If node to this Set node.

5. **Create another GraphQL node:**
   - Name: `Get next page`
   - Similar configuration as first GraphQL node but add `after` variable in query arguments:
     ```graphql
     query ($filter: IssueFilter, $after: String) {
       issues(filter: $filter, first: 100, after: $after) {
         nodes {
           id
           identifier
           url
           title
           priorityLabel
           createdAt
           completedAt
           state {
             type
             name
           }
           cycle {
             number
           }
           estimate
           labels { nodes { name } }
         }
         pageInfo {
           hasNextPage
           endCursor
         }
       }
     }
     ```
   - Variables:
     ```json
     {
       "filter": {
         "team": {
           "name": {
             "eq": "Adore"
           }
         }
       },
       "after": "={{$json.after}}"
     }
     ```
   - Connect Set node output to this GraphQL node input.
   - Connect its output back to the If node to continue pagination loop.

6. **Create a SplitOut node:**
   - Name: `Split out the tickets`
   - Field to split out: `data.issues.nodes`
   - Connect outputs of both GraphQL nodes (first page and next page) to this node.

7. **Create a Set node:**
   - Name: `Set custom fields`
   - Add two fields:
     - `estimate` (Number): `={{ $json.estimate ?? 1 }}`
     - `labels` (String): `={{ $json.labels.nodes.map(label => label.name).join(",") }}`
   - Connect SplitOut node to this node.

8. **Create a Code node:**
   - Name: `Flatten object to have simple fields to filter by`
   - Mode: Run once per item
   - JavaScript code:
     ```javascript
     function flattenObject(ob) {
       let toReturn = {};
       for (let i in ob) {
         if (!ob.hasOwnProperty(i)) continue;
         if ((typeof ob[i]) === 'object' && ob[i] !== null) {
           let flatObject = flattenObject(ob[i]);
           for (let x in flatObject) {
             if (!flatObject.hasOwnProperty(x)) continue;
             toReturn[i + '.' + x] = flatObject[x];
           }
         } else {
           toReturn[i] = ob[i];
         }
       }
       return toReturn;
     }

     return flattenObject($input.item.json);
     ```
   - Connect Set custom fields node to this node.

9. **Create a Google Sheets node:**
   - Name: `Write tickets to Sheets`
   - Operation: appendOrUpdate
   - Mapping mode: autoMapInputData
   - Matching columns: `id`
   - Sheet name: Select your target sheet (e.g., Sheet2)
   - Document ID: Select your Google Sheets document
   - Connect Google Sheets OAuth2 credentials.
   - Connect Code node output to this node.

10. **Add Sticky Note nodes as per your convenience for instructions at these key steps:**
    - Setting team name in GraphQL filter.
    - Adjusting custom fields in Set node.
    - Configuring Google Sheets target.

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow requires a Linear API key set as HTTP header authentication in GraphQL nodes.                                              | Linear API documentation: https://linear.app/docs/graphql-api                                                         |
| Google Sheets node uses OAuth2 credentials; ensure the account has write access to the target spreadsheet.                             | Google Sheets API: https://developers.google.com/sheets/api                                                            |
| The workflow automatically handles pagination to retrieve all tickets beyond the 100-item limit per request.                           | Pagination GraphQL concept: https://graphql.org/learn/pagination/                                                      |
| Only the `id` column is mandatory in the Google Sheet for matching; other columns are auto-created based on ticket data fields.        |                                                                                                |
| Useful for teams not using Linear Plus Insights but needing custom analytical exports or integrations.                                  |                                                                                                                      |
| Workflow screenshot available in the original description for visual reference.                                                         |                                                                                                |