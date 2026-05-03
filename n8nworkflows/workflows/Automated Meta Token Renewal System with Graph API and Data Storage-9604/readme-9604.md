Automated Meta Token Renewal System with Graph API and Data Storage

https://n8nworkflows.xyz/workflows/automated-meta-token-renewal-system-with-graph-api-and-data-storage-9604


# Automated Meta Token Renewal System with Graph API and Data Storage

### 1. Workflow Overview

This workflow automates the renewal of Facebook and Instagram access tokens using Meta’s Graph API and stores updated credentials in a DataTable. It targets use cases where tokens have a limited lifetime (typically 60 days) and require periodic refreshing to maintain uninterrupted API access. The workflow is designed to run on a schedule (every 10 days by default) and renew tokens that are expiring within 15 days, minimizing API calls and credit consumption.

The logical blocks are:

- **1.1 Scheduling and Triggering:** Periodic or manual initiation of the workflow.
- **1.2 Token Retrieval and Expiry Check:** Fetch the current token and its expiration date from a DataTable, then decide if renewal is necessary.
- **1.3 Token Renewal Process:** If renewal is needed, exchange the short-lived token for a new long-lived token via Graph API.
- **1.4 Update Data Storage:** Compute the new expiration date and update the DataTable with the refreshed token and expiry.
- **1.5 No-Op Handling:** Skip renewal if the token is not yet close to expiry.
- **1.6 Documentation and Notes:** Sticky notes provide configuration guidance and operational context.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Triggering

**Overview:**  
Initiates the workflow either on a scheduled basis (every 10 days at 21:00) or manually via a trigger node.

**Nodes Involved:**  
- Schedule Trigger  
- When clicking "Execute workflow"

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically starts the workflow every 10 days at 21:00 to check token expiry.  
  - *Configuration:* Interval trigger set to every 10 days at hour 21.  
  - *Inputs:* None.  
  - *Outputs:* Triggers "Get token expiration date" node.  
  - *Edge cases:* Timing drift or missed executions; adjust schedule as needed to balance token checking frequency and credit usage.

- **When clicking "Execute workflow"**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow for testing or immediate renewal need.  
  - *Inputs:* Manual user action.  
  - *Outputs:* Triggers "Get token expiration date" node.  
  - *Edge cases:* User-triggered, so no timing concerns.

---

#### 1.2 Token Retrieval and Expiry Check

**Overview:**  
Retrieves the current access token and its expiration date from a DataTable, then evaluates if token renewal is required based on expiry proximity.

**Nodes Involved:**  
- Get token expiration date  
- Needs renewal?

**Node Details:**

- **Get token expiration date**  
  - *Type:* Data Table (Get Operation)  
  - *Role:* Fetches the stored token record from the "Meta credential" DataTable.  
  - *Configuration:* Retrieves a single record (limit=1) from the DataTable identified by "TcHt9QIGMyCs1qIB", which holds token and expiry data.  
  - *Inputs:* Trigger from Schedule Trigger or Manual Trigger.  
  - *Outputs:* Passes token data to the "Needs renewal?" node.  
  - *Edge cases:* DataTable empty or inaccessible; failure to retrieve token.

- **Needs renewal?**  
  - *Type:* If Node  
  - *Role:* Determines if the token expires within 15 days from the current date, triggering renewal if true.  
  - *Configuration:* Condition compares the token's `expires_at` date against current date + 15 days. Uses strict type validation with a boolean expression.  
  - *Inputs:* Token record from "Get token expiration date".  
  - *Outputs:*  
    - True branch: proceeds to "Carry ID & Token" for renewal.  
    - False branch: proceeds to "No Operation, do nothing" node.  
  - *Key expression:* `={{ new Date($json.expires_at) <= new Date(Date.now() + 15*24*60*60*1000) }}`  
  - *Edge cases:* Incorrect date format in DataTable, timezone issues, expression evaluation errors.

---

#### 1.3 Token Renewal Process

**Overview:**  
Executes the token exchange with Facebook’s OAuth endpoint to obtain a new long-lived access token.

**Nodes Involved:**  
- Carry ID & Token  
- User Exchange

**Node Details:**

- **Carry ID & Token**  
  - *Type:* Set Node  
  - *Role:* Prepares and passes the current token and record ID forward for the renewal HTTP request.  
  - *Configuration:* Extracts `id` and `token` from previous node JSON and assigns them to `record_id` and `current_token` respectively.  
  - *Inputs:* True branch of "Needs renewal?" node.  
  - *Outputs:* Feeds "User Exchange" node with prepared parameters.  
  - *Edge cases:* Missing or malformed token/id from input JSON.

- **User Exchange**  
  - *Type:* HTTP Request  
  - *Role:* Calls Facebook Graph API to exchange a short-lived token for a long-lived token.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://graph.facebook.com/v20.0/oauth/access_token`  
    - Content-Type: application/x-www-form-urlencoded  
    - Body Parameters:  
      - `grant_type=fb_exchange_token`  
      - `client_id`: Must be set to Facebook App ID (user configured)  
      - `client_secret`: Must be set to Facebook App Secret (user configured)  
      - `fb_exchange_token`: current token from previous node  
  - *Inputs:* From "Carry ID & Token" node.  
  - *Outputs:* Returns JSON containing `access_token` and `expires_in`. Passed to "Compute new expiry".  
  - *Edge cases:* HTTP errors, invalid credentials, rate limiting, expired or invalid tokens.  
  - *Notes:* Requires user to input valid Facebook App credentials.

---

#### 1.4 Update Data Storage

**Overview:**  
Calculates the new token expiry datetime and updates the stored record in the DataTable.

**Nodes Involved:**  
- Compute new expiry  
- Update Record

**Node Details:**

- **Compute new expiry**  
  - *Type:* Set Node  
  - *Role:* Extracts new token and expiry from API response and computes ISO datetime string for expiry.  
  - *Configuration:*  
    - Assigns:  
      - `token` = `access_token` from API response  
      - `expired_at` = Current time + `expires_in` seconds (converted to ISO string)  
      - `record_id` = Carried forward from "Carry ID & Token" node  
  - *Inputs:* From "User Exchange" node.  
  - *Outputs:* Passes updated token and expiry to "Update Record".  
  - *Edge cases:* Missing fields in API response, date calculation errors.

- **Update Record**  
  - *Type:* Data Table (Update Operation)  
  - *Role:* Updates the existing record in "Meta credential" DataTable with the new token and expiry.  
  - *Configuration:*  
    - Matches record by `record_id`  
    - Updates fields:  
      - `token` with new token value  
      - `expires_at` with new ISO expiry date  
  - *Inputs:* From "Compute new expiry".  
  - *Outputs:* None (end of renewal chain).  
  - *Edge cases:* DataTable update failures, concurrency conflicts.

---

#### 1.5 No-Op Handling

**Overview:**  
If the token does not require renewal, the workflow bypasses renewal steps.

**Nodes Involved:**  
- No Operation, do nothing

**Node Details:**

- **No Operation, do nothing**  
  - *Type:* NoOp Node  
  - *Role:* Acts as a pass-through or workflow terminator when renewal is not needed.  
  - *Inputs:* False branch from "Needs renewal?" node.  
  - *Outputs:* None.  
  - *Edge cases:* None (safe operation).

---

#### 1.6 Documentation and Notes

**Overview:**  
Sticky notes provide usage instructions, configuration details, and operational context to assist maintainers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3

**Node Details:**

- **Sticky Note**  
  - *Content Highlights:*  
    - Cron runs every 10 days to limit credit usage; adjust if needed and update renewal logic accordingly.  
    - Facebook tokens currently expire in 60 days.  

- **Sticky Note1**  
  - *Content Highlights:*  
    - Explains DataTable setup: name "Meta credential"  
    - Fields: `token` (string), `expires_at` (datetime)  

- **Sticky Note2**  
  - *Content Highlights:*  
    - Explains token renewal trigger at 15 days before expiry  
    - User must input Facebook `client_id` (App ID) and `client_secret` (App Secret) in "User Exchange" node.  

- **Sticky Note3**  
  - *Content Highlights:*  
    - Details on formatting expiry date from UNIX timestamp to ISO format  
    - Updating the DataTable record with new token and expiry  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                          | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                       |
|---------------------------|---------------------|----------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Starts workflow every 10 days          | —                          | Get token expiration date  | Right now the cron runs every 10 days to avoid consuming too much n8n credits. Feel free to change it. If you change it change also the logic in the "Needs renewal?" node. Facebook tokens have 60 days expiration. |
| When clicking "Execute workflow" | Manual Trigger     | Manual workflow start                   | —                          | Get token expiration date  |                                                                                                |
| Get token expiration date | Data Table (Get)    | Fetch current token and expiry          | Schedule Trigger, Manual Trigger | Needs renewal?             | This workflow is using the new Data tables feature. Create a DataTable "Meta credential" with fields Token (string) and expires_at (datetime). |
| Needs renewal?            | If                  | Check if token expiry < 15 days          | Get token expiration date  | Carry ID & Token (true), No Operation (false) |                                                                                                |
| Carry ID & Token          | Set                  | Prepare token and record ID for renewal | Needs renewal? (true)       | User Exchange              |                                                                                                |
| User Exchange             | HTTP Request         | Exchange current token for new token    | Carry ID & Token           | Compute new expiry          | Generate new tokens if less than 15 days to expiry. Requires setting client_id and client_secret. |
| Compute new expiry        | Set                  | Compute new expiry datetime and pass data | User Exchange              | Update Record              | Setting new expiration date and updating data table: formats date and updates the row.          |
| Update Record             | Data Table (Update) | Update DataTable with new token and expiry | Compute new expiry          | —                          |                                                                                                |
| No Operation, do nothing  | NoOp                 | Skip renewal when not needed             | Needs renewal? (false)      | —                          |                                                                                                |
| Sticky Note               | Sticky Note          | Documentation and notes                  | —                          | —                          | See notes in block 1.6                                                                            |
| Sticky Note1              | Sticky Note          | Documentation on DataTable setup         | —                          | —                          | See notes in block 1.6                                                                            |
| Sticky Note2              | Sticky Note          | Documentation on token renewal config    | —                          | —                          | See notes in block 1.6                                                                            |
| Sticky Note3              | Sticky Note          | Documentation on expiry computation      | —                          | —                          | See notes in block 1.6                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 10 days at 21:00 (hour 21).  
   - Connect its output to “Get token expiration date”.

2. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Connect its output also to “Get token expiration date”.

3. **Create Data Table node - Get token expiration date**  
   - Type: Data Table (Get)  
   - Operation: Get  
   - DataTableId: Create or reference a DataTable named "Meta credential".  
   - Limit: 1 (fetch one record)  
   - Connect inputs from Schedule Trigger and Manual Trigger.  
   - Output connects to "Needs renewal?".

4. **Create If node - Needs renewal?**  
   - Type: If  
   - Condition: Check if `expires_at` ≤ current date + 15 days.  
   - Expression: `={{ new Date($json.expires_at) <= new Date(Date.now() + 15*24*60*60*1000) }}`  
   - True output connects to "Carry ID & Token" node.  
   - False output connects to "No Operation, do nothing".

5. **Create Set node - Carry ID & Token**  
   - Type: Set  
   - Assign:  
     - `record_id` = `{{$json.id}}` (token record ID from previous node)  
     - `current_token` = `{{$json.token}}` (token string)  
   - Connect output to "User Exchange" node.

6. **Create HTTP Request node - User Exchange**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://graph.facebook.com/v20.0/oauth/access_token`  
   - Content-Type: application/x-www-form-urlencoded  
   - Body parameters:  
     - `grant_type` = `fb_exchange_token`  
     - `client_id` = *Enter your Facebook App ID here*  
     - `client_secret` = *Enter your Facebook App Secret here*  
     - `fb_exchange_token` = `={{$json.current_token}}`  
   - Connect output to "Compute new expiry".

7. **Create Set node - Compute new expiry**  
   - Type: Set  
   - Assignments:  
     - `token` = `{{$json.access_token}}` (new token from API response)  
     - `expired_at` = `={{ new Date(Date.now() + ($json.expires_in)*1000).toISOString() }}` (compute new expiry datetime)  
     - `record_id` = `={{ $node["Carry ID & Token"].json.record_id }}` (carry forward record ID)  
   - Connect output to "Update Record".

8. **Create Data Table node - Update Record**  
   - Type: Data Table (Update)  
   - Operation: Update  
   - DataTableId: Same "Meta credential" DataTable as before  
   - Filters: Match record by `id = {{$json.record_id}}`  
   - Columns to update:  
     - `token` = `{{$json.token}}`  
     - `expires_at` = `{{$json.expired_at}}`  
   - Connect output to none (end node).

9. **Create No Operation node - No Operation, do nothing**  
   - Type: NoOp  
   - Connect false branch of "Needs renewal?" here, no further output.

10. **Add Sticky Notes for documentation** (optional but recommended)  
    - Create Sticky Notes with content describing scheduling, DataTable setup, token renewal conditions, and update logic as per original.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                           |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| The workflow uses the new Data Tables feature in n8n; ensure your n8n version supports it.              | n8n official docs on Data Tables                                                                                        |
| Facebook tokens currently have a 60-day expiration window; this workflow renews if less than 15 days remain. | Facebook Graph API Token documentation                                                                                  |
| Adjust schedule trigger interval and renewal threshold (in "Needs renewal?") to balance API usage and token freshness. | Operational tuning advice                                                                                                 |
| Facebook App credentials (`client_id` and `client_secret`) must be securely stored and correctly configured in "User Exchange" node. | Facebook Developer portal                                                                                                |
| This workflow minimizes n8n credit consumption by limiting API calls to every 10 days and renewing only when necessary. | Cost optimization note                                                                                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.