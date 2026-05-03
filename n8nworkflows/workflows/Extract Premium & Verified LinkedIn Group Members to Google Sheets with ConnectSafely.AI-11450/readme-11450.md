Extract Premium & Verified LinkedIn Group Members to Google Sheets with ConnectSafely.AI

https://n8nworkflows.xyz/workflows/extract-premium---verified-linkedin-group-members-to-google-sheets-with-connectsafely-ai-11450


# Extract Premium & Verified LinkedIn Group Members to Google Sheets with ConnectSafely.AI

### 1. Workflow Overview

This workflow is designed to extract members from a specified LinkedIn group using the ConnectSafely.ai API, filter the members for Premium and Verified profiles, and append the filtered data into a Google Sheets spreadsheet. It supports pagination to handle large groups effectively and captures detailed profile information including follower count, headline, and profile URL.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Configuration**: Sets up required credentials, parameters, and pagination variables.
- **1.2 Data Fetching**: Calls the ConnectSafely.ai API to retrieve members in paginated batches.
- **1.3 Filtering & Processing**: Filters members to retain only Premium or Verified profiles and extracts relevant fields.
- **1.4 Pagination Control**: Determines whether to continue fetching more members or proceed to output.
- **1.5 Data Output**: Prepares filtered results and appends them to a Google Sheets spreadsheet.
- **1.6 Completion**: Marks the end of the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Configuration

- **Overview:**  
  This block provides the user with necessary setup instructions, credentials, API details, and pagination initialization.

- **Nodes Involved:**  
  - 📋 Workflow Overview (Sticky Note)  
  - ⚙️ Configuration (Sticky Note)  
  - 🌐 API Details (Sticky Note)  
  - 🔍 Filter Logic (Sticky Note)  
  - 📄 Pagination Flow (Sticky Note)  
  - 📊 Google Sheets (Sticky Note)  
  - Start Workflow (Manual Trigger)  
  - Initialize Pagination (Code)

- **Node Details:**

  - **📋 Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes workflow purpose, features, API endpoint, and author credit.  
    - Key Info: Highlights filtering for Premium & Verified LinkedIn members, pagination, and Google Sheets output.  
    - Input: None  
    - Output: None  

  - **⚙️ Configuration**  
    - Type: Sticky Note  
    - Role: Instructions for setting up API credentials and Google Sheets connection.  
    - Key Info: Requires HTTP Bearer token credential named `ConnectSafelyAI Token` and Google Sheets OAuth2 setup. Default groupId is `9357376`.  
    - Input: None  
    - Output: None  

  - **🌐 API Details**  
    - Type: Sticky Note  
    - Role: Details API endpoint and request/response structure.  
    - Key Info: POST `/linkedin/groups/members` with JSON body containing `groupId`, `count`, and `start`. Response includes `members[]` and `hasMore`.  
    - Input: None  
    - Output: None  

  - **🔍 Filter Logic**  
    - Type: Sticky Note  
    - Role: Defines criteria for filtering LinkedIn members.  
    - Key Info: Filters members where `isPremium === true`, `isVerified === true`, or badges include 'premium'/'verified'. Extracts profile information fields.  
    - Input: None  
    - Output: None  

  - **📄 Pagination Flow**  
    - Type: Sticky Note  
    - Role: Explains pagination logic used in the workflow.  
    - Key Info: Uses `start` offset starting at 0, fetches 50 members per request, loops if `hasMore` is true, else proceeds to output.  
    - Input: None  
    - Output: None  

  - **📊 Google Sheets**  
    - Type: Sticky Note  
    - Role: Describes the Google Sheets output columns and their meaning.  
    - Key Info: Columns include Profile ID, First Name, Last Name, Headline, Profile URL, Follower Count, Is Premium, Is Verified, etc.  
    - Input: None  
    - Output: None  

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters needed.  
    - Input: None  
    - Output: Triggers next node `Initialize Pagination`.  
    - Failure Modes: Manual trigger failure unlikely unless platform issues.  

  - **Initialize Pagination**  
    - Type: Code (JavaScript)  
    - Role: Sets initial pagination parameters and empty array for accumulating members.  
    - Configuration:  
      - `groupId` default is `"9357376"` (Product Hunt Promotion Group)  
      - `count` is 50 (max members per API request)  
      - `start` is 0 (offset)  
      - `hasMore` is true (flag to continue pagination)  
      - `allMembers` is an empty array (to accumulate results)  
    - Input: Trigger from `Start Workflow`  
    - Output: Pagination state JSON to next node  
    - Edge Cases: Misconfiguration of `groupId` or invalid parameters could lead to empty or failed API responses.  

#### 1.2 Data Fetching

- **Overview:**  
  This block performs the API requests to ConnectSafely.ai to fetch group members in batches.

- **Nodes Involved:**  
  - Fetch Group Members (HTTP Request)

- **Node Details:**

  - **Fetch Group Members**  
    - Type: HTTP Request  
    - Role: Calls the ConnectSafely.ai API to fetch LinkedIn group members.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.connectsafely.ai/linkedin/groups/members` (implied from sticky notes)  
      - Authentication: Uses HTTP Bearer token credential `ConnectSafelyAI Token`  
      - Request Body: JSON with `groupId`, `count`, and `start` from input JSON (pagination state)  
      - Headers: Content-Type application/json  
    - Input: Receives pagination state from `Initialize Pagination` or `Continue Pagination` nodes  
    - Output: API response JSON containing `members[]` and `hasMore`  
    - Edge Cases:  
      - Authentication failures (invalid token)  
      - Network timeouts or API downtime  
      - Rate limiting from API provider  
      - Invalid `groupId` leading to empty or error responses  

#### 1.3 Filtering & Processing

- **Overview:**  
  Processes the fetched members data, filters for Premium or Verified members, extracts relevant profile fields, and accumulates results across pages.

- **Nodes Involved:**  
  - Process & Filter Members (Code)  
  - If (Conditional)

- **Node Details:**

  - **Process & Filter Members**  
    - Type: Code (JavaScript)  
    - Role:  
      - Extracts API response and pagination info.  
      - Filters members by Premium/Verified status using `isPremium`, `isVerified`, and badges array.  
      - Maps filtered members to simplified objects with key profile details.  
      - Combines current batch with previously accumulated members.  
      - Sets pagination flags to continue or stop the loop.  
    - Key Expressions/Variables:  
      - Filters where `isPremium === true || isVerified === true || badges includes 'premium' or 'verified'`  
      - Extracted fields: `profileId`, `firstName`, `lastName`, `fullName`, `headline`, `publicIdentifier`, `profileUrl`, `followerCount`, `isPremium`, `isVerified`, `badges`, `relationshipStatus`, `creator`, `fetchedAt`  
      - Pagination fields: `groupId`, `count`, `start`, `hasMore`, `allMembers`, `continueLoop`  
    - Input: API response and previous pagination state  
    - Output: Updated pagination state with filtered members and loop flag  
    - Edge Cases:  
      - Empty `members` array  
      - Missing or malformed fields in member data  
      - Expression or runtime errors if input JSON is malformed  

  - **If**  
    - Type: Conditional (Boolean check)  
    - Role: Checks the `continueLoop` flag to decide whether to continue pagination or proceed to output.  
    - Configuration:  
      - Condition: `continueLoop === false` (Boolean false)  
      - If TRUE (no more pages): routes to `Prepare for Sheets`  
      - If FALSE (more pages): routes to `Continue Pagination`  
    - Input: Output from `Process & Filter Members`  
    - Output: Conditional branches  
    - Edge Cases: Incorrect flag value leading to infinite loops or premature termination  

#### 1.4 Pagination Control

- **Overview:**  
  Manages the iteration over paginated results by updating the pagination state for the next API request.

- **Nodes Involved:**  
  - Continue Pagination (Code)

- **Node Details:**

  - **Continue Pagination**  
    - Type: Code (JavaScript)  
    - Role: Passes updated pagination state to `Fetch Group Members` to request next batch.  
    - Configuration: Copies current pagination state (`groupId`, `count`, `start`, `hasMore`, `allMembers`) unchanged except `start` is incremented in previous node.  
    - Input: Output from `If` node when `continueLoop === true`  
    - Output: Pagination state for next API call  
    - Edge Cases: Potential infinite loops if pagination flags are not updated correctly  

#### 1.5 Data Output

- **Overview:**  
  Converts the accumulated filtered member data into individual items and appends them to a Google Sheets spreadsheet.

- **Nodes Involved:**  
  - Prepare for Sheets (Code)  
  - Append to Google Sheets (Google Sheets)  
  - ✅ Workflow Complete (NoOp)

- **Node Details:**

  - **Prepare for Sheets**  
    - Type: Code (JavaScript)  
    - Role: Splits the combined `allMembers` array into separate items compatible with the Google Sheets node.  
    - Configuration: Returns an empty array if no members found to prevent errors downstream.  
    - Input: Output from `If` node when pagination is complete (`continueLoop === false`)  
    - Output: Individual member items for Google Sheets node  
    - Edge Cases: Empty arrays cause no operation to be performed in Sheets node.  

  - **Append to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends or updates rows in a Google Sheets spreadsheet with member profile data.  
    - Configuration:  
      - Operation: `appendOrUpdate` based on `Profile ID` column to avoid duplicates  
      - Target Sheet: Identified by document ID and GID corresponding to the Product Hunt Promotion Group  
      - Column Mapping: Maps fields such as Profile ID, First Name, Last Name, Headline, Profile URL, Follower Count, Is Premium, Is Verified, Public Identifier, Relationship Status, and Full Name  
      - Credential: Uses `Google Sheets OAuth2` credential  
    - Input: Individual member records prepared by previous node  
    - Output: Success or failure of append/update operation  
    - Edge Cases:  
      - Authentication errors or expired OAuth tokens  
      - Sheet access permission issues  
      - Schema mismatches or missing columns in sheet  
      - API rate limits or quota exceeded  

  - **✅ Workflow Complete**  
    - Type: NoOp  
    - Role: Marks the successful end of the workflow execution.  
    - Input: Output from Google Sheets node  
    - Output: None  
    - Edge Cases: None  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                         |
|----------------------|--------------------|---------------------------------------|-----------------------|------------------------|---------------------------------------------------------------------------------------------------|
| 📋 Workflow Overview | Sticky Note        | Describes workflow purpose and logic  | None                  | None                   | @[youtube](GZ4OrOt3HSY) - LinkedIn Group Members to Google Sheets; Premium & Verified Members Only |
| ⚙️ Configuration      | Sticky Note        | Setup instructions for credentials    | None                  | None                   | API Credentials and Google Sheets setup instructions; default groupId = 9357376                   |
| 🌐 API Details        | Sticky Note        | API endpoint and request/response info| None                  | None                   | Details ConnectSafely.ai API POST `/linkedin/groups/members`                                      |
| 🔍 Filter Logic       | Sticky Note        | Filtering criteria for members        | None                  | None                   | Filters for Premium/Verified members based on flags and badges                                   |
| 📄 Pagination Flow    | Sticky Note        | Explains pagination logic             | None                  | None                   | Pagination uses start offset and count=50; loops while hasMore true                              |
| 📊 Google Sheets      | Sticky Note        | Google Sheets output columns          | None                  | None                   | Lists columns: Profile ID, Name, Headline, URL, Followers, Premium & Verified status              |
| Start Workflow       | Manual Trigger     | Entry point to start workflow          | None                  | Initialize Pagination   |                                                                                                   |
| Initialize Pagination| Code               | Initialize pagination variables        | Start Workflow        | Fetch Group Members     |                                                                                                   |
| Fetch Group Members   | HTTP Request       | Fetch LinkedIn group members           | Initialize Pagination / Continue Pagination | Process & Filter Members |                                                                                                   |
| Process & Filter Members| Code             | Filter members and accumulate results | Fetch Group Members    | If                     |                                                                                                   |
| If                   | Conditional (If)   | Decide to continue pagination or output| Process & Filter Members | Prepare for Sheets / Continue Pagination |                                                                                                   |
| Continue Pagination   | Code               | Update pagination state for next fetch| If (continueLoop=true) | Fetch Group Members     |                                                                                                   |
| Prepare for Sheets    | Code               | Split accumulated members for Sheets   | If (continueLoop=false)| Append to Google Sheets |                                                                                                   |
| Append to Google Sheets| Google Sheets      | Append/update member data in spreadsheet| Prepare for Sheets     | ✅ Workflow Complete    |                                                                                                   |
| ✅ Workflow Complete  | NoOp               | Marks workflow completion              | Append to Google Sheets| None                   |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start Workflow`  
   - Purpose: Manual start for the workflow  

2. **Create Code Node for Pagination Initialization**  
   - Name: `Initialize Pagination`  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     return [
       {
         json: {
           groupId: "9357376",  // Change to your LinkedIn group ID
           count: 50,           // Number of members per API call
           start: 0,            // Pagination start offset
           hasMore: true,
           allMembers: []
         }
       }
     ];
     ```
   - Connect `Start Workflow` → `Initialize Pagination`  

3. **Create HTTP Request Node**  
   - Name: `Fetch Group Members`  
   - Method: POST  
   - URL: `https://api.connectsafely.ai/linkedin/groups/members`  
   - Authentication: HTTP Bearer Credential named `ConnectSafelyAI Token` (setup required in n8n credentials)  
   - Body Parameters (JSON):
     - `groupId`: `={{$json.groupId}}`  
     - `count`: `={{$json.count}}`  
     - `start`: `={{$json.start}}`  
   - Headers: Content-Type `application/json`  
   - Connect `Initialize Pagination` → `Fetch Group Members`  

4. **Create Code Node for Filtering and Pagination Logic**  
   - Name: `Process & Filter Members`  
   - Type: Code (JavaScript)  
   - Code logic:  
     - Extract members and pagination info from API response  
     - Filter members where `isPremium === true || isVerified === true` or badges include 'premium' or 'verified'  
     - Map filtered members to simplified objects with required fields  
     - Append to accumulated members list  
     - Update `start` for next batch if `hasMore` is true, else set `continueLoop` false  
   - Connect `Fetch Group Members` → `Process & Filter Members`  

5. **Create Conditional Node**  
   - Name: `If`  
   - Type: Boolean condition  
   - Condition: Check if `continueLoop === false`  
   - True branch: proceed to prepare data for Google Sheets  
   - False branch: continue pagination  
   - Connect `Process & Filter Members` → `If`  

6. **Create Code Node for Pagination Continuation**  
   - Name: `Continue Pagination`  
   - Type: Code (JavaScript)  
   - Passes updated pagination state for next API call  
   - Connect `If` false output → `Continue Pagination`  
   - Connect `Continue Pagination` → `Fetch Group Members`  

7. **Create Code Node to Prepare Data for Google Sheets**  
   - Name: `Prepare for Sheets`  
   - Type: Code (JavaScript)  
   - Splits `allMembers` array into individual items  
   - Returns empty array if no members  
   - Connect `If` true output → `Prepare for Sheets`  

8. **Create Google Sheets Node**  
   - Name: `Append to Google Sheets`  
   - Operation: `appendOrUpdate`  
   - Document ID: Your target Google Sheets ID  
   - Sheet Name (GID): Target sheet GID  
   - Mapping: Map incoming JSON fields to columns:
     - Profile ID  
     - First Name  
     - Last Name  
     - Full Name  
     - Headline  
     - Public Identifier  
     - Profile URL  
     - Follower Count  
     - Is Premium  
     - Is Verified  
     - Relationship Status  
   - Matching Columns: `Profile ID` for update or append  
   - Credential: Connect your Google Sheets OAuth2 credential  
   - Connect `Prepare for Sheets` → `Append to Google Sheets`  

9. **Create No Operation Node**  
   - Name: `✅ Workflow Complete`  
   - Purpose: Marks workflow execution end  
   - Connect `Append to Google Sheets` → `✅ Workflow Complete`  

10. **Credential Setup**  
    - Create HTTP Bearer credential named `ConnectSafelyAI Token` with your API token  
    - Connect Google Sheets OAuth2 credential for Google Sheets node  

11. **Adjust Parameters**  
    - Modify `groupId` in `Initialize Pagination` code node to target your LinkedIn group  
    - Ensure your Google Sheet has matching columns as specified  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Video overview available: https://youtu.be/GZ4OrOt3HSY                                                        | Workflow overview video on YouTube               |
| Author and API provider: ConnectSafely.ai                                                                     | https://connectsafely.ai                          |
| Default LinkedIn Group used: Product Hunt Promotion Group (ID 9357376)                                         | Can be changed in `Initialize Pagination` node  |
| Google Sheets setup requires matching columns exactly (see sticky note 📊 Google Sheets for column names)       | Ensure spreadsheet columns are created beforehand |
| Pagination supports up to 50 members per request, API may enforce limits                                       | Adjust `count` param carefully                    |
| Filtering uses multiple criteria to detect Premium or Verified status, including badges array                   | See sticky note 🔍 Filter Logic                    |

---

**Disclaimer:** The provided text is generated exclusively from an n8n automation workflow. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.