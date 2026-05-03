Create Snapshot of Contabo VPS instances on a daily basis

https://n8nworkflows.xyz/workflows/create-snapshot-of-contabo-vps-instances-on-a-daily-basis-2403


# Create Snapshot of Contabo VPS instances on a daily basis

---
### 1. Workflow Overview

This workflow automates the daily creation and maintenance of VPS snapshots hosted on Contabo, running automatically every day at midnight. Its core purpose is to secure VPS data by creating fresh snapshots daily while managing snapshot retention by deleting older backups to conserve storage and maintain clarity.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Initialization**: Triggers the workflow every day at midnight, prepares the current date for naming snapshots, and sets credential parameters for API access.

- **1.2 Authentication & UUID Generation**: Authenticates with the Contabo API using OAuth credentials and generates unique UUIDs for request tracking and idempotency headers.

- **1.3 Instance Enumeration**: Retrieves the list of VPS instances from Contabo.

- **1.4 Snapshot Management**: For each instance, lists existing snapshots, determines if a snapshot exists, and based on that either deletes the old snapshot before creating a new one or directly creates a new snapshot.

- **1.5 Cleanup & Snapshot Creation**: Executes deletion of outdated snapshots and issues requests to create new snapshots with uniquely formatted names.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Initialization

**Overview:**  
Initiates the workflow daily at midnight, formats the current date for snapshot labeling, and sets credential parameters needed for Contabo API authentication.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’  
- get Date & Time  
- Formatted Date  
- Credential  
- Sticky Note6  

**Node Details:**  

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts workflow automatically every day at midnight (default daily interval).  
  - *Input:* None  
  - *Output:* Triggers `get Date & Time`.  
  - *Edge Cases:* Timezone mismatch if server timezone differs.  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual testing of the workflow.  
  - *Input:* None  
  - *Output:* Triggers `get Date & Time`.  

- **get Date & Time**  
  - *Type:* DateTime  
  - *Role:* Retrieves current date/time in configured timezone (America/Sao_Paulo).  
  - *Configuration:* Timezone set to America/Sao_Paulo, retries on failure enabled.  
  - *Output:* Provides current date for formatting.  
  - *Edge Cases:* API failure or incorrect timezone setting.  

- **Formatted Date**  
  - *Type:* DateTime  
  - *Role:* Formats the current date into `dd-MM-yyyy` string for snapshot naming.  
  - *Key Expression:* `={{ $json.currentDate }}` input, output formatted string.  
  - *Output:* Formatted date string.  

- **Credential**  
  - *Type:* Set  
  - *Role:* Holds Contabo API credentials (`CLIENT_ID`, `CLIENT_SECRET`, `API_USER`, `API_PASSWORD`).  
  - *Configuration:* User must fill in these credentials before running.  
  - *Output:* Passes credentials to the Authorization node.  

- **Sticky Note6**  
  - *Type:* Sticky Note  
  - *Role:* Provides setup instructions and author credits.  
  - *Content:* Details about credentials and links to author LinkedIn and GitHub.  

---

#### 2.2 Authentication & UUID Generation

**Overview:**  
Authenticates with Contabo’s OAuth endpoint to obtain an access token and generates multiple UUIDs used as unique request identifiers for API calls.

**Nodes Involved:**  
- Authorization  
- UUID  
- TRACE ID  
- UUID1  
- UUID2  
- UUID3  
- UUID4  
- Sticky Note (UUID generator)  

**Node Details:**  

- **Authorization**  
  - *Type:* HTTP Request (POST)  
  - *Role:* Obtains OAuth2 access token from Contabo using password grant type.  
  - *Configuration:* URL is Contabo auth token endpoint; sends `client_id`, `client_secret`, `username`, `password` as x-www-form-urlencoded body parameters.  
  - *Output:* Provides `access_token` and `token_type` for subsequent API calls.  
  - *Edge Cases:* Authentication failure due to wrong credentials, network timeout.  

- **UUID, TRACE ID, UUID1, UUID2, UUID3, UUID4**  
  - *Type:* HTTP Request (GET)  
  - *Role:* Each node requests a new UUID v4 from `https://www.uuidgenerator.net/api/version4`.  
  - *Purpose:* These UUIDs are used for `x-request-id` and `x-trace-id` HTTP headers to ensure idempotency and traceability per API call.  
  - *Edge Cases:* API unavailability or rate limiting from UUID generator.  

- **Sticky Note1**  
  - *Content:* Explains the purpose of UUID generation with a link to the UUID API.  

---

#### 2.3 Instance Enumeration

**Overview:**  
Fetches the list of all VPS instances available under the authenticated Contabo account.

**Nodes Involved:**  
- List instances  
- Split Out  
- Sticky Note2  

**Node Details:**  

- **List instances**  
  - *Type:* HTTP Request (GET)  
  - *Role:* Calls Contabo API `/v1/compute/instances` endpoint to retrieve instances list.  
  - *Headers:* Includes `Authorization` with bearer token, `Content-Type: application/json`, and `x-request-id`/`x-trace-id` set with UUID and TRACE ID generated previously.  
  - *Output:* JSON array of VPS instance metadata.  
  - *Edge Cases:* API errors, invalid token, or empty instance list.  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Separates the array of instances into individual items for processing.  
  - *Configuration:* Splits on the `data` field.  
  - *Output:* Individual instance JSON objects forwarded to next nodes.  

- **Sticky Note2**  
  - *Content:* Briefly describes that this node lists all VPS instances.  

---

#### 2.4 Snapshot Management

**Overview:**  
For each VPS instance, checks for existing snapshots, and determines whether to delete old snapshots before creating new ones or just create fresh snapshots if none exist.

**Nodes Involved:**  
- List snapshots  
- Whether snapshot there is no snapshot (If)  
- Set snapshot attributes (two variants)  
- Merge  
- Sticky Note3  
- Sticky Note4  

**Node Details:**  

- **List snapshots**  
  - *Type:* HTTP Request (GET)  
  - *Role:* Retrieves snapshots for the current instance using `/v1/compute/instances/{id}/snapshots`.  
  - *Headers:* Similar to instance listing but uses new UUIDs for `x-request-id` and `x-trace-id`.  
  - *Input:* Instance ID from `Split Out` node.  
  - *Output:* Snapshot list for the instance.  
  - *Edge Cases:* API failure, empty snapshot array.  

- **Whether snapshot there is no snapshot**  
  - *Type:* If  
  - *Role:* Evaluates if the snapshot list is empty (no snapshot exists).  
  - *Condition:* Checks if snapshot data array is empty.  
  - *Outputs:*  
    - *True:* No snapshot exists → proceeds to create snapshot directly.  
    - *False:* Snapshot(s) found → proceeds to delete existing snapshot before creating new one.  

- **Set snapshot attributes (two nodes)**  
  - *Type:* Set  
  - *Role:* Extracts and sets attributes like `instanceId`, `displayName`, and `snapshotId` for downstream usage.  
  - *Note:* One node (`set snapshot attributes`) handles creation path, the other (`Set snapshot attributes`) handles deletion + creation path.  
  - *Edge Cases:* Missing fields in input JSON may cause expression errors.  

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines outputs from listing snapshots with the split instance data to prepare for conditional processing.  
  - *Mode:* Combine by position ensures corresponding instance and snapshot data are aligned.  

- **Sticky Notes3 & 4**  
  - *Content:* Describe the snapshot listing process and the deletion step respectively.  

---

#### 2.5 Cleanup & Snapshot Creation

**Overview:**  
Deletes old snapshots if present, then creates new snapshots for VPS instances using formatted date labels and instance display names.

**Nodes Involved:**  
- Delete existing snapshot  
- Create a new snapshot  
- Create a new snapshot1  
- UUID2, UUID3, UUID4 (UUID generators for these operations)  
- Sticky Note5  

**Node Details:**  

- **Delete existing snapshot**  
  - *Type:* HTTP Request (DELETE)  
  - *Role:* Deletes the old snapshot by snapshot ID for the given instance.  
  - *URL:* Uses instanceId and snapshotId from `Set snapshot attributes`.  
  - *Headers:* Includes authorization and unique UUIDs for `x-request-id` and `x-trace-id`.  
  - *Retry:* Enabled to handle transient failures.  
  - *Output:* Confirms deletion before next step.  
  - *Edge Cases:* Snapshot not found, permission errors, network timeouts.  

- **Create a new snapshot & Create a new snapshot1**  
  - *Type:* HTTP Request (POST)  
  - *Role:* Creates a fresh snapshot for the instance with a name formatted as the current date and a description combining the instance display name and date.  
  - *Headers:* Same authorization and UUID headers.  
  - *Retry:* Enabled for robustness.  
  - *Note:* Two separate nodes exist, one for creation after deletion, one for creation when no snapshot existed.  
  - *Edge Cases:* API limits, invalid data, network issues.  

- **UUID2, UUID3, UUID4**  
  - *Role:* Generate unique IDs for each API call above ensuring request uniqueness.  

- **Sticky Note5**  
  - *Content:* Explains the snapshot creation process.  

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                              | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                                  |
|-----------------------------|----------------------|----------------------------------------------|--------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger     | Starts workflow daily at midnight             | None                           | get Date & Time                    |                                                                                                              |
| When clicking ‘Test workflow’ | Manual Trigger      | Manual workflow start for testing             | None                           | get Date & Time                    |                                                                                                              |
| get Date & Time             | DateTime             | Gets current datetime in configured timezone | Schedule Trigger, Manual Trigger | Formatted Date                   |                                                                                                              |
| Formatted Date              | DateTime             | Formats date for snapshot naming               | get Date & Time                | Credential                        |                                                                                                              |
| Credential                 | Set                  | Stores Contabo API credentials                 | Formatted Date                 | Authorization                    | ## Credential Information required (CLIENT_ID, CLIENT_SECRET, API_USER, API_PASSWORD) [Contabo Credential](https://my.contabo.com/api/details) [Contabo API Doc](https://api.contabo.com/) |
| Authorization              | HTTP Request         | Authenticates with Contabo API                 | Credential                    | UUID                            |                                                                                                              |
| UUID                       | HTTP Request         | Generates UUID for request headers             | Authorization                | TRACE ID                        | ## get UUID Generates the UUIDs that will be used in the 'x-request-id' and 'x-trace-id' [uuidgenerator](https://www.uuidgenerator.net/api) |
| TRACE ID                   | HTTP Request         | Generates UUID for request headers             | UUID                         | List instances                  |                                                                                                              |
| List instances             | HTTP Request         | Lists VPS instances                             | TRACE ID                     | Split Out                      | ## List your instances                                                                                         |
| Split Out                  | Split Out            | Splits instance list into individual objects   | List instances               | UUID1, Merge                   |                                                                                                              |
| UUID1                      | HTTP Request         | Generates UUID for snapshot list requests      | Split Out                    | List snapshots                 |                                                                                                              |
| List snapshots             | HTTP Request         | Lists snapshots for each VPS instance          | UUID1                        | Merge                         | ## List existing Snapshots - Generates a new UUID for the request - Checks if the instance already has a Snapshot |
| Merge                      | Merge                | Combines instance and snapshot data            | List snapshots, Split Out     | Whether snapshot there is no snapshot |                                                                                                              |
| Whether snapshot there is no snapshot | If               | Checks if there are snapshots                    | Merge                        | set snapshot attributes, Set snapshot attributes |                                                                                                              |
| set snapshot attributes    | Set                  | Sets instance attributes for snapshot creation | Whether snapshot (true)       | UUID2                        |                                                                                                              |
| Set snapshot attributes    | Set                  | Sets instance and snapshot IDs for deletion    | Whether snapshot (false)      | UUID3                        | ## Delete existing snapshot by id                                                                             |
| UUID2                      | HTTP Request         | Generates UUID for create snapshot request     | set snapshot attributes       | Create a new snapshot          |                                                                                                              |
| Create a new snapshot      | HTTP Request         | Creates snapshot when no previous snapshot     | UUID2                        |                                | ## Create a new snapshot                                                                                       |
| UUID3                      | HTTP Request         | Generates UUID for delete snapshot request     | Set snapshot attributes       | Delete existing snapshot       |                                                                                                              |
| Delete existing snapshot   | HTTP Request         | Deletes old snapshot                            | UUID3                        | UUID4                        |                                                                                                              |
| UUID4                      | HTTP Request         | Generates UUID for creating new snapshot after deletion | Delete existing snapshot | Create a new snapshot1          |                                                                                                              |
| Create a new snapshot1     | HTTP Request         | Creates new snapshot after deleting old one    | UUID4                        |                                |                                                                                                              |
| Sticky Note6               | Sticky Note          | Setup instructions and credits                  | None                        | None                         | ## Contabo Backups Workflow Setup instructions and author credits with links                                  |
| Sticky Note1               | Sticky Note          | Explains UUID generation                         | None                        | None                         | ## get UUID [uuidgenerator](https://www.uuidgenerator.net/api)                                                |
| Sticky Note2               | Sticky Note          | Describes instance listing                       | None                        | None                         | ## List your instances                                                                                         |
| Sticky Note3               | Sticky Note          | Describes snapshot listing                       | None                        | None                         | ## List existing Snapshots                                                                                     |
| Sticky Note4               | Sticky Note          | Describes snapshot deletion                       | None                        | None                         | ## Delete existing snapshot by id                                                                             |
| Sticky Note5               | Sticky Note          | Describes snapshot creation                       | None                        | None                         | ## Create a new snapshot                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger daily at midnight (default daily interval).  

2. **Add a Manual Trigger node**  
   - For manual workflow testing.  

3. **Add a DateTime node (`get Date & Time`)**  
   - Configure timezone to `America/Sao_Paulo`.  
   - Connect Schedule Trigger and Manual Trigger to this node.  

4. **Add another DateTime node (`Formatted Date`)**  
   - Operation: Format Date  
   - Input: `={{ $json.currentDate }}` from `get Date & Time` node  
   - Format: Custom `dd-MM-yyyy`  
   - Connect `get Date & Time` node to this node.  

5. **Add a Set node (`Credential`)**  
   - Define string fields: `CLIENT_ID`, `CLIENT_SECRET`, `API_USER`, `API_PASSWORD`.  
   - Leave values blank for user to fill.  
   - Connect `Formatted Date` node to this node.  

6. **Add an HTTP Request node (`Authorization`)**  
   - Method: POST  
   - URL: `https://auth.contabo.com/auth/realms/contabo/protocol/openid-connect/token`  
   - Headers: `Content-Type: application/x-www-form-urlencoded`  
   - Body Parameters (x-www-form-urlencoded):  
     - `client_id`: `={{ $json.CLIENT_ID }}`  
     - `client_secret`: `={{ $json.CLIENT_SECRET }}`  
     - `username`: `={{ $json.API_USER }}`  
     - `password`: `={{ $json.API_PASSWORD }}`  
     - `grant_type`: `password`  
   - Connect `Credential` node to this node.  

7. **Add UUID generator HTTP Request nodes (six in total):**  
   - For each (`UUID`, `TRACE ID`, `UUID1`, `UUID2`, `UUID3`, `UUID4`):  
     - Method: GET  
     - URL: `https://www.uuidgenerator.net/api/version4`  
   - Connect nodes in the following order:  
     - `Authorization` → `UUID`  
     - `UUID` → `TRACE ID`  
     - `Split Out` → `UUID1`  
     - `set snapshot attributes` → `UUID2`  
     - `Set snapshot attributes` → `UUID3`  
     - `Delete existing snapshot` → `UUID4`  

8. **Add HTTP Request node (`List instances`)**  
   - Method: GET  
   - URL: `https://api.contabo.com/v1/compute/instances`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Authorization`: `={{ $('Authorization').item.json['token_type'] }} {{ $('Authorization').item.json['access_token'] }}`  
     - `x-request-id`: `={{ $('UUID').item.json['data'] }}`  
     - `x-trace-id`: `={{ $('TRACE ID').item.json['data'] }}`  
   - Connect `TRACE ID` node to this node.  

9. **Add Split Out node (`Split Out`)**  
   - Configure to split on `data` field.  
   - Connect `List instances` node to this node.  

10. **Add HTTP Request node (`List snapshots`)**  
    - Method: GET  
    - URL: `=https://api.contabo.com/v1/compute/instances/{{ $('Split Out').item.json['instanceId'] }}/snapshots`  
    - Headers:  
      - `Content-Type`: `application/json`  
      - `Authorization`: `={{ $('Authorization').item.json['token_type'] }} {{ $('Authorization').item.json['access_token'] }}`  
      - `x-request-id`: `={{ $('UUID1').item.json['data'] }}`  
      - `x-trace-id`: `={{ $('TRACE ID').item.json['data'] }}`  
    - Connect `UUID1` node to this node.  

11. **Add Merge node (`Merge`)**  
    - Mode: Combine  
    - Combine by position  
    - Connect `Split Out` to second input, `List snapshots` to first input.  

12. **Add If node (`Whether snapshot there is no snapshot`)**  
    - Condition: Check if snapshot list (`$('List snapshots').item.json['data']`) is empty.  
    - Connect `Merge` node to this node.  

13. **Add two Set nodes:**  
    - `set snapshot attributes` (for no snapshot path)  
      - Assign `instanceId` and `displayName` from input JSON.  
    - `Set snapshot attributes` (for existing snapshot path)  
      - Assign `instanceId`, `displayName`, and `data[0].snapshotId` from input JSON for deletion.  
    - Connect If node outputs:  
      - True → `set snapshot attributes`  
      - False → `Set snapshot attributes`  

14. **Add HTTP Request node (`Delete existing snapshot`)**  
    - Method: DELETE  
    - URL: `=https://api.contabo.com/v1/compute/instances/{{ $('Set snapshot attributes').item.json['instanceId'] }}/snapshots/{{ $('Set snapshot attributes').item.json['data'][0].snapshotId }}`  
    - Headers:  
      - `Content-Type`: `application/json`  
      - `Authorization`: `={{ $('Authorization').item.json['token_type'] }} {{ $('Authorization').item.json['access_token'] }}`  
      - `x-request-id`: `={{ $('UUID3').item.json['data'] }}`  
      - `x-trace-id`: `={{ $('TRACE ID').item.json['data'] }}`  
    - Retry on fail enabled.  
    - Connect `UUID3` to this node.  

15. **Add two HTTP Request nodes for snapshot creation:**  
    - `Create a new snapshot` (when no snapshot previously exists)  
      - Method: POST  
      - URL: `=https://api.contabo.com/v1/compute/instances/{{ $('set snapshot attributes').item.json['instanceId'] }}/snapshots`  
      - Headers:  
        - `Content-Type`: `application/json`  
        - `Authorization`: `={{ $('Authorization').item.json['token_type'] }} {{ $('Authorization').item.json['access_token'] }}`  
        - `x-request-id`: `={{ $('UUID2').item.json['data'] }}`  
        - `x-trace-id`: `={{ $('TRACE ID').item.json['data'] }}`  
      - Body Parameters:  
        - `name`: `={{ $('Formatted Date').item.json['formattedDate'] }}`  
        - `description`: `={{ $('set snapshot attributes').item.json['displayName'] }} {{ $('Formatted Date').item.json['formattedDate'] }}`  
      - Retry on fail enabled.  
      - Connect `UUID2` node to this node.  

    - `Create a new snapshot1` (after deleting old snapshot)  
      - Same configuration as above but uses `Set snapshot attributes` node data and UUID4 for headers.  
      - Connect `UUID4` node to this node.  
      - Connect `Delete existing snapshot` node to `UUID4`.  

16. **Add Sticky Notes**  
    - Add notes as per the original workflow to provide context and instructions.

17. **Activate the workflow**  
    - Ensure credentials are filled before activation.  
    - Test with manual trigger to verify correct operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow created by Marcos Antonio. LinkedIn: https://www.linkedin.com/in/compromitto/ GitHub: https://github.com/dubcom 🇧🇷       | Author credits                                                                                              |
| Contabo API Credential setup and documentation: https://my.contabo.com/api/details and https://api.contabo.com/                   | Credential configuration and API reference                                                                 |
| UUID Generation API: https://www.uuidgenerator.net/api                                                                              | Used to generate unique UUID v4 strings for request headers                                                |
| Workflow runs daily at midnight, timezone configured as America/Sao_Paulo                                                          | Ensure server timezone consistency or adjust accordingly                                                   |

---

This comprehensive documentation enables understanding, modification, and accurate reproduction of the “Create Snapshot of Contabo VPS instances on a daily basis” workflow in n8n. It anticipates potential errors, such as authentication failures, API timeouts, or missing snapshot data, while providing clear instructions and references for setup and operation.