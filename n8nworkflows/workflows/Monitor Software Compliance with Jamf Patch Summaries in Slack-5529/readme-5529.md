Monitor Software Compliance with Jamf Patch Summaries in Slack

https://n8nworkflows.xyz/workflows/monitor-software-compliance-with-jamf-patch-summaries-in-slack-5529


# Monitor Software Compliance with Jamf Patch Summaries in Slack

### 1. Workflow Overview

This workflow automates monitoring software compliance through Jamf Patch Management and posts summarized status reports into a Slack channel. It is designed for IT administrators or compliance officers who need to track patch levels of specific software titles managed by Jamf and notify teams via Slack.

The workflow is structured into these logical blocks:

- **1.1 Initialization & Configuration**: Set Jamf server URL and trigger workflow manually.
- **1.2 Fetch Software Titles**: Retrieve a list of all software titles configured in Jamf Patch Management.
- **1.3 Filter Software**: Select specific software titles by their IDs that are being tracked.
- **1.4 Retrieve Patch Summaries**: For each selected software, fetch the detailed patch summary.
- **1.5 Data Formatting**: Map and format the data into Slack Block Kit structure.
- **1.6 Post to Slack**: Send the formatted patch compliance summary to a Slack channel.

The workflow uses OAuth2 authentication to communicate securely with Jamf Cloud APIs and Slack API for messaging.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Configuration

- **Overview:**  
Sets the Jamf server hostname for subsequent API calls and provides a manual trigger to start the workflow. Includes sticky notes for user guidance.

- **Nodes Involved:**  
  - Click (Manual Trigger)  
  - Jamf Server (Set Node)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **Click**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Jamf Server node  
    - Edge Cases: None  

  - **Jamf Server**  
    - Type: Set  
    - Role: Defines the Jamf server hostname to build API URLs dynamically.  
    - Configuration: Sets the string variable `server` with a placeholder value `"yourServer"` to be replaced with actual Jamf Cloud server name.  
    - Inputs: Manual Trigger  
    - Outputs: Get Software Title/ID node  
    - Edge Cases: Server name must be set correctly; otherwise, API calls will fail with connection errors.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides instructions to set the Jamf Base URL.  
    - Content:  
      ```
      ## Set  
      Set your jamf BaseURL 
      from your jamf URL
      https://yourServer.jamfcloud.com
      ```
    - Placement: Near Jamf Server node.

---

#### 2.2 Fetch Software Titles

- **Overview:**  
Retrieves all software titles configured in Jamf Patch Management using the Jamf API.

- **Nodes Involved:**  
  - Get Software Title/ID (HTTP Request)  
  - Sticky Note1 (Instructional)

- **Node Details:**

  - **Get Software Title/ID**  
    - Type: HTTP Request  
    - Role: Retrieves software title configurations from Jamf Patch Management API.  
    - Configuration:  
      - URL: `https://{{ $json.server }}.jamfcloud.com/api/v2/patch-software-title-configurations`  
      - Authentication: OAuth2 via genericCredentialType (configured with Jamf OAuth2 credentials)  
      - Method: GET (default)  
    - Inputs: Jamf Server (provides server variable)  
    - Outputs: ID List node  
    - Edge Cases:  
      - API authentication failure (expired tokens, invalid credentials)  
      - Network timeouts  
      - API rate limiting  
      - Empty or malformed responses  

  - **Sticky Note1**  
    - Content:  
      ```
      ## Get  
      Get all list of software from your jamf patch management
      ```

---

#### 2.3 Filter Software

- **Overview:**  
Filters the list of software titles to only those specified by their IDs, representing the software products currently tracked.

- **Nodes Involved:**  
  - ID List (Filter)  
  - Sticky Note2 (Instructional)

- **Node Details:**

  - **ID List**  
    - Type: Filter  
    - Role: Selects only software titles with specified IDs from the fetched list.  
    - Configuration:  
      - Conditions: OR logic matching `id` field equals to either `<software_id>` or `<software_id>` (placeholders for actual software IDs).  
      - Case sensitive and strict type validation enabled.  
    - Inputs: Get Software Title/ID  
    - Outputs: Get Patch Summary:ID node  
    - Edge Cases:  
      - No matching IDs → downstream steps may receive no input and produce empty outputs.  
      - Typo or incorrect IDs → missing expected software entries.

  - **Sticky Note2**  
    - Content:  
      ```
      ## Filter
      Filter to select the software you're tracking - add
      ```
    - Note: Incomplete sentence, likely indicating user must add software IDs.

---

#### 2.4 Retrieve Patch Summaries

- **Overview:**  
For each filtered software title, retrieves detailed patch summary data including latest version and compliance statistics.

- **Nodes Involved:**  
  - Get Patch Summary:ID (HTTP Request)  
  - Sticky Note3 (Instructional)

- **Node Details:**

  - **Get Patch Summary:ID**  
    - Type: HTTP Request  
    - Role: Fetches patch summary for each software title by ID from Jamf API.  
    - Configuration:  
      - URL: `https://{{ $('Jamf Server').item.json.server }}.jamfcloud.com/api/v2/patch-software-title-configurations/{{ $json.id }}/patch-summary`  
      - Authentication: OAuth2 (same as previous)  
    - Inputs: ID List (filtered software IDs)  
    - Outputs: Manual Field Mapping node  
    - Edge Cases:  
      - API errors if software ID invalid or deleted  
      - Authentication failures  
      - Network issues  

  - **Sticky Note3**  
    - Content:  
      ```
      ## Retrieve
      Retrieve the patch summary for the selected softwares
      ```

---

#### 2.5 Data Formatting

- **Overview:**  
Maps the raw patch summary data fields into simplified keys and formats the entire dataset into Slack Block Kit JSON for rich message formatting.

- **Nodes Involved:**  
  - Manual Field Mapping (Set)  
  - Block Formatting (Code)  
  - Sticky Note4 (Instructional)  
  - Sticky Note6 (Instructional)

- **Node Details:**

  - **Manual Field Mapping**  
    - Type: Set  
    - Role: Extracts and renames fields from Jamf API response into simplified keys for easier formatting.  
    - Configuration:  
      - Fields mapped:  
        - `Title` ← `$json.title`  
        - `Latest version` ← `$json.latestVersion`  
        - `up To Date` ← `$json.upToDate` (number)  
        - `out Of Date` ← `$json.outOfDate` (number)  
        - `Jamf application url` ← constructed URL using server and softwareTitleId  
    - Inputs: Get Patch Summary:ID  
    - Outputs: Block Formatting  
    - Edge Cases:  
      - Missing fields in API response  
      - Null or unexpected data types  

  - **Block Formatting**  
    - Type: Code (JavaScript)  
    - Role: Constructs Slack Block Kit JSON message to present patch status clearly.  
    - Configuration:  
      - Builds a header block with title "📋 Patch Management Status"  
      - Iterates over input items, adding a section block per software containing title, latest version, compliance counts, and a link.  
      - Inserts dividers between sections, removing the last divider.  
      - Returns JSON with `blocks` key for Slack message.  
    - Inputs: Manual Field Mapping  
    - Outputs: Slack Channel  
    - Edge Cases:  
      - Empty input array → message contains only header  
      - Malformed or missing fields cause formatting issues or runtime errors  

  - **Sticky Note4**  
    - Content:  
      ```
      ## Format
      Format the summary into Slack Block Kit
      ```

  - **Sticky Note6**  
    - Content:  
      ```
      ## Select
      Select only the needed data
      ```

---

#### 2.6 Post to Slack

- **Overview:**  
Sends the formatted patch management summary to a designated Slack channel via Slack API using Block Kit message format.

- **Nodes Involved:**  
  - Slack Channel (Slack Node)  
  - Sticky Note5 (Instructional)

- **Node Details:**

  - **Slack Channel**  
    - Type: Slack  
    - Role: Posts the patch summary blocks to a Slack channel as a block message.  
    - Configuration:  
      - Message Type: Block  
      - Channel ID: Configured via credentials or UI (empty in JSON, must be set)  
      - Blocks UI: Uses expression to insert blocks from previous node  
      - Credential: Slack API OAuth token configured externally  
    - Inputs: Block Formatting  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Invalid Slack token → authentication failure  
      - Invalid or missing channel ID → message posting fails  
      - Slack rate limiting or API errors  

  - **Sticky Note5**  
    - Content:  
      ```
      ## Post
      Post the formatted summary into a Slack channel
      ```

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                                   | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                 |
|------------------------|--------------------|-------------------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| Click                  | Manual Trigger     | Entry point to start the workflow                | None                     | Jamf Server              |                                                                                             |
| Jamf Server            | Set                | Set Jamf server hostname variable                 | Click                    | Get Software Title/ID    | ## Set  Set your jamf BaseURL from your jamf URL https://yourServer.jamfcloud.com          |
| Sticky Note            | Sticky Note        | Instruction to set Jamf Base URL                   | None                     | None                     | ## Set  Set your jamf BaseURL from your jamf URL https://yourServer.jamfcloud.com          |
| Get Software Title/ID  | HTTP Request       | Retrieve all software titles from Jamf             | Jamf Server              | ID List                  | ## Get  Get all list of software from your jamf patch management                           |
| Sticky Note1           | Sticky Note        | Instructional note for fetching software titles   | None                     | None                     | ## Get  Get all list of software from your jamf patch management                           |
| ID List                | Filter             | Filter software titles to those tracked            | Get Software Title/ID    | Get Patch Summary:ID     | ## Filter Filter to select the software you're tracking - add                              |
| Sticky Note2           | Sticky Note        | Instructional note for filtering                    | None                     | None                     | ## Filter Filter to select the software you're tracking - add                              |
| Get Patch Summary:ID   | HTTP Request       | Get patch summary details for each software        | ID List                  | Manual Field Mapping     | ## Retrieve Retrieve the patch summary for the selected softwares                         |
| Sticky Note3           | Sticky Note        | Instructional note for retrieving patch summaries  | None                     | None                     | ## Retrieve Retrieve the patch summary for the selected softwares                         |
| Manual Field Mapping   | Set                | Map and rename fields from API response             | Get Patch Summary:ID     | Block Formatting         | ## Select Select only the needed data                                                     |
| Sticky Note6           | Sticky Note        | Instructional note for field selection              | None                     | None                     | ## Select Select only the needed data                                                     |
| Block Formatting       | Code               | Format data into Slack Block Kit JSON                | Manual Field Mapping     | Slack Channel            | ## Format Format the summary into Slack Block Kit                                         |
| Sticky Note4           | Sticky Note        | Instructional note for formatting                    | None                     | None                     | ## Format Format the summary into Slack Block Kit                                         |
| Slack Channel          | Slack              | Post the formatted summary message into Slack      | Block Formatting         | None                     | ## Post Post the formatted summary into a Slack channel                                  |
| Sticky Note5           | Sticky Note        | Instructional note for posting                        | None                     | None                     | ## Post Post the formatted summary into a Slack channel                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `Click`.

2. **Add a Set node** named `Jamf Server`:  
   - Connect it from `Click`.  
   - Add a string field `server` with value set to your Jamf Cloud server hostname (e.g., `"yourServer"`).

3. **Add an HTTP Request node** named `Get Software Title/ID`:  
   - Connect from `Jamf Server`.  
   - Set URL to:  
     `https://{{ $json.server }}.jamfcloud.com/api/v2/patch-software-title-configurations`  
   - Authentication: OAuth2 (configure Jamf OAuth2 credentials in n8n).  
   - Method: GET.

4. **Add a Filter node** named `ID List`:  
   - Connect from `Get Software Title/ID`.  
   - Configure filter with OR conditions:  
     - `id` equals `<software_id_1>`  
     - `id` equals `<software_id_2>`  
   - Replace `<software_id_1>`, `<software_id_2>` with actual software IDs you want to track.

5. **Add an HTTP Request node** named `Get Patch Summary:ID`:  
   - Connect from `ID List`.  
   - URL:  
     `https://{{ $('Jamf Server').item.json.server }}.jamfcloud.com/api/v2/patch-software-title-configurations/{{ $json.id }}/patch-summary`  
   - Use same OAuth2 credentials.  
   - Method: GET.

6. **Add a Set node** named `Manual Field Mapping`:  
   - Connect from `Get Patch Summary:ID`.  
   - Assign fields:  
     - `Title` = `{{$json.title}}`  
     - `Latest version` = `{{$json.latestVersion}}`  
     - `up To Date` = `{{$json.upToDate}}` (number)  
     - `out Of Date` = `{{$json.outOfDate}}` (number)  
     - `Jamf application url` = `https://{{ $('Jamf Server').item.json.server }}.jamfcloud.com/view/computers/patch/{{ $json.softwareTitleId }}`

7. **Add a Code node** named `Block Formatting`:  
   - Connect from `Manual Field Mapping`.  
   - Use the following JavaScript code:  
     ```javascript
     const inputItems = $input.all();
     const blocks = [
       {
         type: "header",
         text: { type: "plain_text", text: "📋 Patch Management Status" }
       }
     ];
     inputItems.forEach(item => {
       const browser = item.json;
       blocks.push({
         type: "section",
         text: {
           type: "mrkdwn",
           text: `*${browser.Title}*\n• Latest version: \`${browser["Latest version"]}\`\n• ✅ Up to date: *${browser["up To Date"]}*\n• ⚠️ Out of date: *${browser["out Of Date"]}*\n• 🔗 URL: ${browser["Jamf application url"]}`
         }
       });
       blocks.push({ type: "divider" });
     });
     blocks.pop(); // Remove last divider
     return [{ json: { blocks } }];
     ```

8. **Add a Slack node** named `Slack Channel`:  
   - Connect from `Block Formatting`.  
   - Set Message Type to `Block`.  
   - For blocks UI, use expression:  
     `={{ '{ "blocks": [' + JSON.stringify($json.blocks).slice(1, -1) + '] }' }}`  
   - Set the Slack channel ID where the message should be posted.  
   - Configure Slack OAuth2 credentials with proper permissions (`chat:write`).

9. **Test the workflow manually** by clicking the `Click` manual trigger and verify messages are posted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow requires Jamf Cloud OAuth2 credentials setup in n8n for API authentication.                 | Jamf Cloud API documentation: https://developer.jamf.com/     |
| Slack node requires Slack App OAuth token with `chat:write` scope configured in n8n credentials.    | Slack API docs: https://api.slack.com/authentication/oauth-v2 |
| Sticky notes in the workflow provide inline user instructions for configuration and operation.      | Workflow UI                                                      |
| Replace placeholder software IDs in the Filter node with actual Jamf software title IDs to track.    | Obtain from Jamf web console or API                             |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.