Automatically Join WhatsApp Groups with Google Sheets and Evolution API

https://n8nworkflows.xyz/workflows/automatically-join-whatsapp-groups-with-google-sheets-and-evolution-api-8880


# Automatically Join WhatsApp Groups with Google Sheets and Evolution API

### 1. Workflow Overview

This workflow automates joining WhatsApp groups using invitation codes stored in a Google Sheets spreadsheet. It is designed to:

- Periodically read invitation codes from a Google Sheet.
- Process the first 50 unused invitation codes per run.
- Validate each invitation code by fetching group details through the Evolution API.
- Attempt to join the WhatsApp groups corresponding to valid invitation codes.
- Update the Google Sheet with the status of each join attempt (success or failure).
- Log successful joins into a tracking list within Google Sheets.
- Implement wait periods to avoid hitting API usage limits.

**Logical blocks:**

- **1.1 Scheduling & Input Acquisition:** Automatically triggers the workflow at scheduled times and reads invitation codes from Google Sheets.
- **1.2 Preprocessing:** Filters the first 50 unused invitation codes and prepares data for processing.
- **1.3 Group Validation:** Uses the Evolution API to validate invitation codes and fetch group details.
- **1.4 Decision & Joining:** Decides whether to join a group based on validation; attempts to join using the Evolution API.
- **1.5 Status Update & Logging:** Updates Google Sheets with join statuses and appends successful joins to a tracking list.
- **1.6 Throttling:** Introduces wait times between batches to manage rate limits and API quotas.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Input Acquisition

**Overview:**  
This block triggers the workflow at specified times and reads invitation codes from a Google Sheet to start the process.

**Nodes Involved:**  
- Schedule Trigger  
- Lire invitation code (Google Sheets read)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow automatically at specific times (6:08, 12:13, and 20:17).  
  - Configuration: Three daily trigger times set.  
  - Inputs: None (start node).  
  - Outputs: Triggers "Lire invitation code".  
  - Edge Cases: Workflow will not run outside scheduled times.  
  - Sticky Note: “Runs automatically at the frequency you set (e.g., daily, hourly).”

- **Lire invitation code**  
  - Type: Google Sheets (Read operation)  
  - Role: Reads invitation codes from a configured Google Sheet.  
  - Configuration: Document ID and sheet name specified (placeholders in JSON).  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Outputs rows containing invitation codes and associated data.  
  - Credentials: Google Sheets OAuth2 API credentials required.  
  - Edge Cases:  
    - Authentication errors if credentials expire.  
    - Empty sheet or missing columns will cause incomplete data.  
  - Sticky Note: “Reads invitation codes from the Google Sheet.”

---

#### 1.2 Preprocessing

**Overview:**  
Filters the first 50 unused invitation codes and splits them into batches for iterative processing.

**Nodes Involved:**  
- 50 premiers non traités (Code node)  
- Loop Over Items (SplitInBatches)  

**Node Details:**

- **50 premiers non traités**  
  - Type: Code Node (JavaScript)  
  - Role: Keeps only the first 50 invitation codes from the input list.  
  - Configuration: `return items.slice(0, 50);`  
  - Inputs: Rows from Google Sheets node.  
  - Outputs: Maximum of 50 items for further processing.  
  - Edge Cases: If fewer than 50 codes are available, processes all available.  
  - Failure: The JS code is simple; failure unlikely unless input is undefined.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Splits the 50 invitation codes into batches of 1 (default) for sequential processing.  
  - Inputs: Filtered list from Code node.  
  - Outputs: Single item per batch for processing.  
  - Edge Cases: Empty input causes no iterations.  
  - Failure: None expected.

---

#### 1.3 Group Validation

**Overview:**  
Validates each invitation code by fetching group details using the Evolution API.

**Nodes Involved:**  
- Mapper données (Set node)  
- Fetch groups (Evolution API node)  
- Merge (Merge node)

**Node Details:**

- **Mapper données**  
  - Type: Set  
  - Role: Maps and assigns key data fields (`row_number`, `Invitation Code`, `Groupe id`) for use downstream.  
  - Inputs: Individual batch item from SplitInBatches.  
  - Outputs: Prepared JSON object with mapped fields.  
  - Edge Cases: Missing expected fields in input JSON may cause empty values downstream.

- **Fetch groups**  
  - Type: Evolution API (Fetch Groups operation)  
  - Role: Validates the invitation code by fetching group details.  
  - Configuration: Uses the invitation code from the previous node.  
  - Inputs: Mapped data from "Mapper données".  
  - Outputs: Group details including group size.  
  - Credentials: Evolution API credentials required.  
  - Edge Cases:  
    - Invitation code invalid or expired results in no or error response.  
    - API timeout or authentication failure.  
  - On error: Set to continue with regular output to avoid workflow stop.

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from "Mapper données" and "Fetch groups" nodes by position to aggregate all relevant data per item.  
  - Inputs: Two inputs — mapped data and fetched group details.  
  - Outputs: Combined data for decision-making.  
  - Edge Cases: Mismatched input lengths could cause misaligned data.

---

#### 1.4 Decision & Joining

**Overview:**  
Decides if the group is appropriate to join based on the group size, attempts to join if criteria met.

**Nodes Involved:**  
- If (Conditional)  
- Join group (Evolution API)

**Node Details:**

- **If**  
  - Type: If conditional node  
  - Role: Checks if group size is greater than 50 to decide whether to join the group.  
  - Configuration: Condition: `$json.data.size > 50` (strict type validation).  
  - Inputs: Merged data combining invitation code and group info.  
  - Outputs: True branch proceeds to "Join group"; false branch proceeds to status update without joining.  
  - Edge Cases: Missing or malformed group size will cause condition to fail or skip joining.

- **Join group**  
  - Type: Evolution API (Join Group operation)  
  - Role: Attempts to join the WhatsApp group using the invitation code.  
  - Inputs: Invitation code from "Mapper données".  
  - Outputs: Join result (success or failure).  
  - Credentials: Evolution API credentials required.  
  - On error: Continue with regular output to avoid workflow stop.  
  - Edge Cases:  
    - Invalid or expired invite code causes failure.  
    - API limits or authentication errors.  
  - Sticky Note: Explains joining attempt and logging behavior.

---

#### 1.5 Status Update & Logging

**Overview:**  
Updates the Google Sheet with the join status and appends successful joins to a tracking list.

**Nodes Involved:**  
- Mis a jour statut (Google Sheets update)  
- Remplir liste (Google Sheets append)  
- Mis a jour statut1 (Google Sheets update)

**Node Details:**

- **Mis a jour statut**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the status of the join attempt in the main invitation codes sheet.  
  - Inputs: Join group output.  
  - Configuration: Document ID and sheet name specified (placeholders).  
  - Credentials: Google Sheets OAuth2 API credentials.  
  - Edge Cases:  
    - Update failures if the sheet is locked or API limit reached.  
    - Incorrect document or sheet name causes update failure.

- **Remplir liste**  
  - Type: Google Sheets (Append operation)  
  - Role: Records successful joins by appending to a tracking list or log sheet.  
  - Inputs: Status update output.  
  - Configuration: Document ID and sheet name specified (placeholders).  
  - Credentials: Google Sheets OAuth2 API credentials.  
  - Edge Cases: Append failures due to API limits or incorrect sheet.

- **Mis a jour statut1**  
  - Type: Google Sheets (Update operation)  
  - Role: Updates the status of invitation codes that were not joined (based on "If" false branch).  
  - Inputs: From "If" node false branch.  
  - Configuration: Document ID and sheet name specified (placeholders).  
  - Credentials: Google Sheets OAuth2 API credentials.  
  - Edge Cases: As above, update may fail due to API or sheet issues.

---

#### 1.6 Throttling

**Overview:**  
Implements wait times to avoid hitting API limits before processing the next invitation code batch.

**Nodes Involved:**  
- Wait

**Node Details:**

- **Wait**  
  - Type: Wait node  
  - Role: Pauses execution briefly before continuing to the next batch of invitation codes.  
  - Inputs: From status update nodes.  
  - Outputs: Loops back to "Loop Over Items" for next batch processing.  
  - Configuration: Default wait (duration not explicitly set in JSON).  
  - Edge Cases: Excessive wait may delay processing; no wait might trigger API throttling.  
  - Sticky Note: Notes about using short wait to avoid limits.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                  |
|-----------------------|----------------------------------|--------------------------------------------|------------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Triggers workflow on schedule               | None                   | Lire invitation code    | “Runs automatically at the frequency you set (e.g., daily, hourly).”                                        |
| Lire invitation code  | Google Sheets (Read)             | Reads invitation codes from Google Sheet    | Schedule Trigger       | 50 premiers non traités | “Reads invitation codes from the Google Sheet.”                                                             |
| 50 premiers non traités | Code (JavaScript)               | Selects first 50 unused invitation codes    | Lire invitation code   | Loop Over Items         | “Filters to only process the first 50 unused invitation codes.”                                             |
| Loop Over Items       | SplitInBatches                   | Processes invitation codes one by one       | 50 premiers non traités | Mapper données (on second output branch) |                                                                                                              |
| Mapper données        | Set                             | Maps relevant fields for downstream use     | Loop Over Items        | Fetch groups, Merge     |                                                                                                              |
| Fetch groups          | Evolution API (Fetch Groups)     | Validates invitation code and fetches group details | Mapper données         | Merge                  | “Validates the invitation code and fetches group details.”                                                  |
| Merge                 | Merge                           | Combines mapped data and group details      | Mapper données, Fetch groups | If                   |                                                                                                              |
| If                    | If                              | Checks if group size > 50 to decide on joining | Merge                  | Join group (true), Mis a jour statut1 (false) |                                                                                                              |
| Join group            | Evolution API (Join Group)       | Attempts to join WhatsApp group              | If (true branch)       | Mis a jour statut       | “Try to join the WhatsApp group. If successful, update the sheet and add to the joined list. If failed, mark the code as failed.” |
| Mis a jour statut     | Google Sheets (Update)           | Updates join status in main sheet            | Join group             | Remplir liste           |                                                                                                              |
| Remplir liste         | Google Sheets (Append)           | Logs successful joins in tracking list       | Mis a jour statut      | Wait                   | “Log results back into Google Sheets and use a short wait to avoid hitting limits before processing the next code.” |
| Mis a jour statut1    | Google Sheets (Update)           | Updates status for codes not joined          | If (false branch)      | Wait                   |                                                                                                              |
| Wait                  | Wait                            | Waits briefly before processing next batch   | Mis a jour statut1, Remplir liste | Loop Over Items  | “Use a short wait to avoid API limits.”                                                                      |
| Sticky Note           | Sticky Note                     | Notes and explanations                        | None                   | None                   | Multiple notes explaining workflow blocks and node purposes.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set trigger times at 06:08, 12:13, and 20:17 daily.

2. **Add a Google Sheets node (Lire invitation code)**  
   - Operation: Read Rows  
   - Configure with your Google Sheets document ID and sheet name that holds the invitation codes.  
   - Use Google Sheets OAuth2 credentials with read access.

3. **Add a Code node (50 premiers non traités)**  
   - JavaScript code: `return items.slice(0, 50);`  
   - Connect the output of Google Sheets read node to this node.

4. **Add a SplitInBatches node (Loop Over Items)**  
   - Default batch size (1).  
   - Connect output of Code node to this node.

5. **Add a Set node (Mapper données)**  
   - Assign the following fields from input JSON:  
     - `row_number` (number)  
     - `Invitation Code` (string)  
     - `Groupe id` (string)  
   - Connect second output of Loop Over Items to this node.

6. **Add an Evolution API node (Fetch groups)**  
   - Resource: groups-api  
   - Operation: fetch-groups  
   - Parameter: Use expression to get invitation code from `Mapper données` node: `{{ $json["Invitation Code"] }}`  
   - Use Evolution API credentials with appropriate permissions.  
   - Connect output of Mapper données to this node.

7. **Add a Merge node**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect output of Mapper données to first input; output of Fetch groups to second input.

8. **Add an If node**  
   - Condition: Check if `$json.data.size` (from fetched group info) is greater than 50.  
   - Strict type validation enabled.  
   - Connect output of Merge node to this node.

9. **Add an Evolution API node (Join group)**  
   - Resource: groups-api  
   - Operation: join-group  
   - Parameter: Use expression to get invitation code from `Mapper données`: `{{ $json["Invitation Code"] }}`  
   - Use Evolution API credentials.  
   - Connect True output of If node to this node.

10. **Add a Google Sheets node (Mis a jour statut)**  
    - Operation: Update  
    - Configure with document ID and sheet name where status is updated.  
    - Connect output of Join group node to this node.

11. **Add a Google Sheets node (Remplir liste)**  
    - Operation: Append  
    - Configure with document ID and sheet name for logging successful joins.  
    - Connect output of Mis a jour statut node to this node.

12. **Add a Google Sheets node (Mis a jour statut1)**  
    - Operation: Update  
    - Configure with document ID and sheet name to update status for codes not joined (If node False branch).  
    - Connect False output of If node to this node.

13. **Add a Wait node**  
    - Default wait time or customize as needed (to avoid API rate limits).  
    - Connect output of Remplir liste and Mis a jour statut1 to this node.

14. **Connect output of Wait node back to Loop Over Items node**  
    - This creates a loop processing batches sequentially with pauses.

15. **Credentials Setup:**  
    - Configure Google Sheets OAuth2 credentials with both read and write access to the respective sheets.  
    - Configure Evolution API credentials with permissions for fetching group info and joining groups.

16. **Test the workflow:**  
    - Use a few sample invitation codes in your Google Sheet.  
    - Run the workflow manually or wait for the scheduled trigger.  
    - Verify status updates and logs in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Auto-join WhatsApp groups from Google Sheets invitation codes. This workflow automates joining groups, updating statuses, and tracking results without manual intervention. It requires setting up Google Sheets with invitation codes and using the Evolution API for WhatsApp group operations. Testing is recommended before scaling.                                                                                                                                                                                                                                     | Workflow description embedded in Sticky Note3 node.                                                                 |
| Evolution API used for WhatsApp integration is self-hosted only; ensure your instance is properly configured with credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                       | Evolution API node credential setup.                                                                                  |
| Schedule Trigger enables automation at chosen frequencies (e.g., daily at specific times). Adjust to your needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Schedule Trigger node configuration.                                                                                  |
| Google Sheets API credentials must allow read and write operations on the specified spreadsheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Google Sheets OAuth2 credentials.                                                                                      |
| Use wait nodes responsibly to avoid hitting API rate limits or quotas, especially when processing many invitation codes.                                                                                                                                                                                                                                                                                                                                                                                                                                            | Wait node usage notes.                                                                                                |
| More information on Evolution API and community nodes for WhatsApp integration can be found on n8n forums and official documentation.                                                                                                                                                                                                                                                                                                                                                                                                                                | n8n community and Evolution API references.                                                                            |

---

This document fully describes the workflow for automatic joining of WhatsApp groups using invitation codes managed in Google Sheets. It covers all nodes, logic, and setup details necessary for reproduction and modification.