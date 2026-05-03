Sync Productboard Features to Linear Issues with Telegram Notifications

https://n8nworkflows.xyz/workflows/sync-productboard-features-to-linear-issues-with-telegram-notifications-8891


# Sync Productboard Features to Linear Issues with Telegram Notifications

### 1. Workflow Overview

This workflow automates the synchronization of new features from Productboard into Linear as issues, with real-time success notifications sent via Telegram. Its primary use case is to keep Linear issue tracking aligned with Productboard’s feature roadmap, ensuring that newly created Productboard features are promptly reflected as issues in Linear for the product and development teams. The workflow runs on a schedule and is structured into four logical blocks:

- **1.1 Trigger & Data Retrieval:** Periodically triggers the workflow and pulls feature data from Productboard via API.
- **1.2 Data Transformation:** Cleans and restructures the raw Productboard data into a simplified, consistent format.
- **1.3 Filtering New Features:** Filters features to process only those created on the current day, avoiding duplicates.
- **1.4 Issue Creation & Notifications:** Creates corresponding Linear issues for new features and sends Telegram notifications confirming the sync.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Retrieval

- **Overview:**  
This block initiates the workflow on a scheduled basis and fetches the complete list of features from Productboard using authenticated HTTP requests.

- **Nodes Involved:**  
  - 📅 Schedule Trigger  
  - 🌐 HTTP Request to ProductBoard  

- **Node Details:**

  - **📅 Schedule Trigger**  
    - *Type & Role:* Schedule trigger node that starts the workflow according to a predefined interval.  
    - *Configuration:* Default interval set (empty object implies standard scheduling, typically daily).  
    - *Input/Output:* No input; outputs trigger signal to HTTP Request to ProductBoard.  
    - *Edge Cases:* Misconfiguration can lead to no triggers or too frequent triggers; ensure correct timezone if relevant.

  - **🌐 HTTP Request to ProductBoard**  
    - *Type & Role:* HTTP Request node to retrieve features from Productboard’s REST API.  
    - *Configuration:*  
      - URL: `https://api.productboard.com/features`  
      - Headers include Authorization Bearer token (replace `YOUR_PRODUCTBOARD_API_TOKEN` with valid token), Content-Type JSON, and API version header `X-Version: 1`.  
      - Method defaults to GET (implied).  
    - *Input/Output:* Receives trigger from Schedule Trigger; outputs raw Productboard JSON features data.  
    - *Edge Cases:*  
      - API token invalid or expired → authentication errors.  
      - Network timeouts or rate limits.  
      - Productboard API changes or downtime.

---

#### 2.2 Data Transformation

- **Overview:**  
Transforms the raw JSON response from Productboard into a cleaned, simplified JSON array with plain text descriptions and standardized date formats.

- **Nodes Involved:**  
  - 💻 Code (Transform Features)  

- **Node Details:**

  - **💻 Code (Transform Features)**  
    - *Type & Role:* JavaScript Code node to process and normalize feature data.  
    - *Configuration:*  
      - Extracts key fields: id, name, type, status (name), parent feature/component ID, owner email, plain text description (HTML tags stripped), created and updated dates formatted as `YYYY-MM-DD`, and feature link URL.  
      - Uses `.map()` to iterate features and clean data accordingly.  
      - Strips HTML tags from description using regex.  
    - *Key Expressions:*  
      - `d.description.replace(/<[^>]*>/g, '').trim()` for description cleaning.  
      - Date formatting via `new Date(...).toISOString().split("T")[0]`.  
    - *Input/Output:* Input is raw JSON from Productboard node; output is array of feature objects, each as an individual item.  
    - *Edge Cases:*  
      - Missing or null fields handled with fallback to `null`.  
      - Description may contain unexpected HTML formats that regex might not fully clean.  
      - Date fields malformed or missing → result in `null`.

---

#### 2.3 Filtering New Features

- **Overview:**  
Filters the transformed features to only those created on the current date, preventing reprocessing of older features.

- **Nodes Involved:**  
  - ⚖️ If (Filter New Features)  

- **Node Details:**

  - **⚖️ If (Filter New Features)**  
    - *Type & Role:* Conditional node to filter items based on creation date.  
    - *Configuration:*  
      - Condition compares `createdAt` field of each feature to today’s date (computed dynamically inside expression).  
      - Uses strict equality, case sensitive, string type validation.  
    - *Key Expressions:*  
      - Left: `{{$json.createdAt}}`  
      - Right: `{{ new Date($json.createdAt).toISOString().split("T")[0] }}` (evaluates to current date in UTC format)  
    - *Input/Output:* Input: cleaned features; Output: only features created today proceed to issue creation; others are discarded.  
    - *Edge Cases:*  
      - Timezone differences may cause mismatch in "today" calculation depending on server time vs. local time.  
      - Features missing `createdAt` will fail the condition and be excluded.  
      - If workflow is triggered multiple times a day, features created earlier that day will be reprocessed unless additional logic is added.

---

#### 2.4 Issue Creation & Notifications

- **Overview:**  
Creates new issues in Linear for each filtered feature and sends Telegram notifications upon success.

- **Nodes Involved:**  
  - 📝 Create Linear Issue  
  - 📢 Success Notification (Telegram)  

- **Node Details:**

  - **📝 Create Linear Issue**  
    - *Type & Role:* Linear node to create an issue in a specified Linear team.  
    - *Configuration:*  
      - Title set to feature `name`.  
      - Team ID specified by user (`YOUR_LINEAR_TEAM_ID`).  
      - Description set to feature `description`.  
      - Credentials must be configured with a valid Linear API token.  
    - *Input/Output:* Input: filtered new features; Output: new Linear issue objects passed to Telegram node.  
    - *Edge Cases:*  
      - Invalid or expired Linear credentials cause authentication failure.  
      - Rate limiting or API downtime.  
      - Missing required fields in Linear (e.g. teamId) will cause errors.

  - **📢 Success Notification (Telegram)**  
    - *Type & Role:* Telegram node to send a text message to a chat.  
    - *Configuration:*  
      - Message text dynamically includes synced feature name from the If node’s JSON.  
      - Uses configured Telegram bot credentials.  
      - Chat ID set to target Telegram group or user.  
    - *Key Expressions:*  
      - `=✅ Successfully synced Productboard feature "{{ $('⚖️ If (Filter New Features)').item.json.name }}" with Linear`  
    - *Input/Output:* Input: output from Linear issue creation; output is Telegram message sent (no further node).  
    - *Edge Cases:*  
      - Invalid Telegram bot token or chat ID results in failures.  
      - Network or Telegram API downtime.

---

### 3. Summary Table

| Node Name                     | Node Type                    | Functional Role                          | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                                                                         |
|-------------------------------|------------------------------|----------------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 📅 Schedule Trigger            | Schedule Trigger              | Starts workflow on schedule             | None                            | 🌐 HTTP Request to ProductBoard  | ## 🗓 Trigger & Data Fetching *Schedule Trigger starts the workflow at defined intervals. HTTP Request (Productboard) fetches all features via API.*              |
| 🌐 HTTP Request to ProductBoard | HTTP Request                | Fetches Productboard features           | 📅 Schedule Trigger             | 💻 Code (Transform Features)     | See above                                                                                                                                                         |
| 💻 Code (Transform Features)  | Code                         | Cleans and normalizes Productboard data | 🌐 HTTP Request to ProductBoard | ⚖️ If (Filter New Features)       | ## 🔄 Data Transformation *Strips HTML from descriptions, extracts key fields, formats dates as YYYY-MM-DD.*                                                     |
| ⚖️ If (Filter New Features)    | If                           | Filters features created today          | 💻 Code (Transform Features)   | 📝 Create Linear Issue            | ## ✅ Filtering Logic *Filters only newly created features to avoid duplicates.*                                                                                  |
| 📝 Create Linear Issue         | Linear                       | Creates issues in Linear for new features | ⚖️ If (Filter New Features)     | 📢 Success Notification (Telegram) | ## 📝 Issue Creation & Notifications *Creates Linear issues and sends Telegram confirmation messages.*                                                           |
| 📢 Success Notification (Telegram) | Telegram                 | Sends notification on successful sync   | 📝 Create Linear Issue          | None                             | See above                                                                                                                                                         |
| Sticky Note                   | Sticky Note                   | Documentation                          | None                            | None                             | ## 🗓 Trigger & Data Fetching *Schedule Trigger and HTTP Request nodes explanation.*                                                                               |
| Sticky Note1                  | Sticky Note                   | Documentation                          | None                            | None                             | ## 🔄 Data Transformation *Code node explanation for cleaning Productboard data.*                                                                                 |
| Sticky Note2                  | Sticky Note                   | Documentation                          | None                            | None                             | ## ✅ Filtering Logic *If node explanation for filtering new features only.*                                                                                      |
| Sticky Note3                  | Sticky Note                   | Documentation                          | None                            | None                             | ## 📝 Issue Creation & Notifications *Linear issue creation and Telegram notification explanation.*                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to desired frequency (e.g., daily at midnight). Leave default interval if daily trigger is acceptable.  
   - Position: Start node of the workflow.

2. **Add HTTP Request node to fetch Productboard features**  
   - Type: HTTP Request  
   - Connect from Schedule Trigger output.  
   - Set URL to `https://api.productboard.com/features`.  
   - Add headers:  
     - `Authorization: Bearer YOUR_PRODUCTBOARD_API_TOKEN` (replace with actual token)  
     - `Content-Type: application/json`  
     - `X-Version: 1`  
   - Use GET method (default).  
   - No body parameters needed.

3. **Add Code node to transform features**  
   - Type: Code  
   - Connect from HTTP Request node output.  
   - Paste the following JavaScript code to:  
     - Extract relevant fields (id, name, type, status name, parentId, owner email, description stripped from HTML, createdAt and updatedAt dates formatted as `YYYY-MM-DD`, and feature link).  
     - Strip HTML tags from description using regex.  
     - Map each feature to a clean JSON object and return as separate items.

4. **Add If node to filter new features**  
   - Type: If  
   - Connect from Code node output.  
   - Configure condition:  
     - Left value: `{{$json.createdAt}}`  
     - Operator: Equals  
     - Right value: `{{ new Date($json.createdAt).toISOString().split("T")[0] }}` (current date in UTC)  
   - This ensures only features created today proceed.

5. **Add Linear node to create issues**  
   - Type: Linear  
   - Connect from If node’s "true" output.  
   - Configure:  
     - Title: `={{ $json.name }}`  
     - Team ID: `YOUR_LINEAR_TEAM_ID` (replace with actual team ID)  
     - Additional fields: Description set to `={{ $json.description }}`  
   - Set Linear credentials with a valid API token.

6. **Add Telegram node for success notification**  
   - Type: Telegram  
   - Connect from Linear node output.  
   - Configure:  
     - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (target chat/group ID)  
     - Text: `=✅ Successfully synced Productboard feature "{{ $('⚖️ If (Filter New Features)').item.json.name }}" with Linear`  
   - Set Telegram credentials with valid bot token.

7. **Verify all connections:**  
   - Schedule Trigger → HTTP Request to Productboard  
   - HTTP Request → Code (Transform Features)  
   - Code → If (Filter New Features)  
   - If (true) → Linear issue creation  
   - Linear → Telegram notification

8. **Test the workflow:**  
   - Run manually or wait for scheduled trigger.  
   - Confirm features created today in Productboard generate new Linear issues and Telegram messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Productboard API documentation: https://developer.productboard.com/reference/get-features                                | Reference for API usage and authentication                                                         |
| Linear API documentation: https://developers.linear.app/docs/graphql/getting-started                                      | Required for setting up credentials and issue fields                                                |
| Telegram Bot API: https://core.telegram.org/bots/api                                                                      | For Telegram credential setup and chat ID retrieval                                                 |
| This workflow assumes all date comparisons use UTC timezone, consider adapting if local timezone logic is required.       | Important for accurate filtering of newly created features                                          |
| Ensure API tokens and credentials are kept secure and permissions allow reading features and creating issues.             | Security best practice                                                                                |
| The description cleaning via regex removes HTML tags but may not handle complex HTML or embedded content perfectly.       | May require enhancement if rich text descriptions are used                                         |

---

**Disclaimer:** The provided text results exclusively from an automated workflow developed with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.