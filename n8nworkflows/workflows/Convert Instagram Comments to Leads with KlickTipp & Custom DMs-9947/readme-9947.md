Convert Instagram Comments to Leads with KlickTipp & Custom DMs

https://n8nworkflows.xyz/workflows/convert-instagram-comments-to-leads-with-klicktipp---custom-dms-9947


# Convert Instagram Comments to Leads with KlickTipp & Custom DMs

---
### 1. Workflow Overview

This workflow automates the process of converting Instagram post comments into marketing leads by integrating Instagram, Google Sheets, and KlickTipp CRM via n8n. It listens for new Instagram comments, validates webhook setup challenges, filters comments based on keywords, sends personalized direct messages (DMs) with lead capture form links, and manages lead data in a Google Sheets matching table for further processing in KlickTipp.

The workflow is logically divided into the following blocks:

- **1.1 Data Reception**: Receiving Instagram comment webhook events and validating webhook subscription challenges.
- **1.2 Validation and Message Sending**: Filtering comments for keywords and sending personalized DMs with a lead capture form.
- **1.3 Data Storage and Matching**: Checking and updating the Google Sheets matching table to track lead information.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Reception

- **Overview:**  
  This block receives incoming Instagram webhook events for new comments and handles the initial webhook verification challenge sent by Meta to confirm the webhook subscription.

- **Nodes Involved:**  
  - Listen to new Instagram comments  
  - Check first validation  
  - Challenge to validate  
  - Sticky Note (labeling this block)

- **Node Details:**

  1. **Listen to new Instagram comments**  
     - Type: Webhook  
     - Role: Entry point listening for Instagram comment events from Meta’s webhook.  
     - Configuration:  
       - Webhook path uniquely identified.  
       - Response mode set to respond via a response node.  
     - Inputs: External HTTP webhook calls from Meta (Instagram).  
     - Outputs: Passes webhook JSON payload to validation node.  
     - Edge cases: Potential webhook failures if Meta cannot reach the URL, signature verification errors (not explicitly handled here).  
     - Notes: Requires the Meta app to be configured with this webhook URL.

  2. **Check first validation**  
     - Type: Switch  
     - Role: Determines if the incoming request is a webhook verification challenge or an actual Instagram comment event.  
     - Configuration:  
       - Checks if `hub.verify_token` query parameter equals the expected token `KlickTipp`.  
       - Branches into either “Initial Webhook verification” or “Processing branch after setup.”  
     - Inputs: JSON from webhook node.  
     - Outputs: Either routes to response node for verification or to comment processing nodes.  
     - Edge cases: Failure to validate token leads to ignoring the request; no explicit error response.

  3. **Challenge to validate**  
     - Type: Respond to Webhook  
     - Role: Replies to Meta’s webhook verification challenge by echoing the `hub.challenge` value.  
     - Configuration: Responds with plain text containing `hub.challenge` query parameter.  
     - Inputs: From “Initial Webhook verification” branch.  
     - Outputs: Ends workflow execution for verification requests.  
     - Edge cases: Missing `hub.challenge` would cause a blank response—should be avoided by Meta standards.

  4. **Sticky Note**  
     - Content: “## 1. Data reception” to label this logical block visually.

---

#### 1.2 Validation and Message Sending

- **Overview:**  
  This block checks if the Instagram comment contains a specific keyword, and if so, sends a personalized DM containing a link to a lead capture form.

- **Nodes Involved:**  
  - Does comment contain keyword?  
  - DM form to user  
  - Sticky Note1  
  - Sticky Note2 (providing community node disclaimer and detailed project description)

- **Node Details:**

  1. **Does comment contain keyword?**  
     - Type: If  
     - Role: Filters incoming comments to process only those containing the keyword "keyword".  
     - Configuration:  
       - Checks if the comment text (`value.text`) contains the string `"keyword"`.  
       - Case-sensitive, strict string contains operation.  
     - Inputs: Instagram comment JSON.  
     - Outputs: Passes comments containing the keyword to the DM sending node.  
     - Edge cases: Comments without the keyword are dropped silently.

  2. **DM form to user**  
     - Type: HTTP Request  
     - Role: Sends a direct message (DM) to the Instagram user who commented, prompting them to fill out a form.  
     - Configuration:  
       - POST request to Instagram Graph API endpoint for sending messages.  
       - Uses HTTP Header Authentication with provided Instagram credentials.  
       - Message body personalized with the user’s Instagram username and user ID extracted from the webhook payload.  
       - Message text includes a greeting, explanation, and a dynamic link to a JotForm form prefilled with the Instagram username.  
     - Inputs: Filtered comment containing the keyword.  
     - Outputs: Passes data onwards to the data storage block.  
     - Edge cases: Possible failures include API auth errors, rate limits, invalid recipient ID, or malformed message JSON.

  3. **Sticky Note1**  
     - Content: “## 2. Validate and send message” to label this block.

  4. **Sticky Note2**  
     - Content: Detailed community node disclaimer and extensive project description including use cases, setup instructions, and customization tips.  
     - Covers the entire workflow context and target audience.

---

#### 1.3 Data Storage and Matching

- **Overview:**  
  This block checks if the commenting user already exists in a Google Sheets matching table and adds them if not present. This table acts as a lead tracking database facilitating later KlickTipp integration.

- **Nodes Involved:**  
  - Look for entry in matching table  
  - Does entry exist?  
  - Add entry to matching table  
  - Sticky Note3

- **Node Details:**

  1. **Look for entry in matching table**  
     - Type: Google Sheets  
     - Role: Searches for an existing row with the Instagram username in the Google Sheet.  
     - Configuration:  
       - Filters rows where “Instagram username” column matches the username from the webhook payload.  
       - Reads from the first sheet (gid=0) of a specific Google Sheets document.  
       - Uses OAuth2 credentials for Google Sheets.  
     - Inputs: Data passed from the DM sending node.  
     - Outputs: Passes matching row info to the next decision node.  
     - Edge cases: Google API errors, permission issues, or sheet not found.

  2. **Does entry exist?**  
     - Type: If  
     - Role: Checks if the lookup returned an existing row number, meaning the user is already tracked.  
     - Configuration:  
       - Checks if `row_number` exists in the lookup result.  
     - Inputs: Result from Google Sheets lookup.  
     - Outputs:  
       - If exists: ends or continues (no further action defined here).  
       - If not exists: routes to add new entry node.  
     - Edge cases: False negatives if no row number returned; no explicit error handling.

  3. **Add entry to matching table**  
     - Type: Google Sheets  
     - Role: Adds a new row with the Instagram username and ID to the matching table.  
     - Configuration:  
       - Appends a row with two columns: "Instagram username" and "Instagram ID comment payload."  
       - Values are dynamically set from the webhook payload.  
       - Uses same Google Sheets document and OAuth2 credentials as lookup.  
     - Inputs: From "Does entry exist?" if user not found.  
     - Outputs: Workflow ends or continues downstream (not shown).  
     - Edge cases: Google API quota exceeded, network issues.

  4. **Sticky Note3**  
     - Content: “## 3. Save data” to label this block.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                                | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------------|-------------------------|-----------------------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Listen to new Instagram comments | Webhook                 | Entry point receiving Instagram comment webhooks | (external webhook calls)      | Check first validation          |                                                                                                |
| Check first validation          | Switch                  | Validates webhook subscription or routes event | Listen to new Instagram comments | Challenge to validate, Does comment contain keyword? | Checks whether there is a first validation challenge that needs to be responded to               |
| Challenge to validate           | Respond to Webhook      | Responds to Meta webhook verification challenge | Check first validation (Initial Webhook verification) | (end)                        | Responds to the webhook and sends back the value for the challenge                              |
| Does comment contain keyword?   | If                      | Filters comments to those containing the keyword | Check first validation (Processing branch after setup) | DM form to user                | Checks if the comment included the intended keyword                                            |
| DM form to user                 | HTTP Request            | Sends personalized DM with lead capture form  | Does comment contain keyword?  | Look for entry in matching table | Sends out a DM to user on Instagram prompting form submission                                  |
| Look for entry in matching table | Google Sheets           | Checks if user already exists in matching table | DM form to user               | Does entry exist?               | Checks if there is an entry in the matching table for this username                            |
| Does entry exist?               | If                      | Determines if user is already tracked          | Look for entry in matching table | (none if exists), Add entry to matching table (if not) | Checks whether the entry already exists in the matching table                                  |
| Add entry to matching table     | Google Sheets           | Adds new Instagram user entry to matching table | Does entry exist? (no branch) | (end)                         | Adds an entry to the matching table                                                           |
| Sticky Note                    | Sticky Note             | Labels block 1.1 Data Reception                 |                              |                               | ## 1. Data reception                                                                           |
| Sticky Note1                   | Sticky Note             | Labels block 1.2 Validation and message sending |                              |                               | ## 2. Validate and send message                                                                |
| Sticky Note2                   | Sticky Note             | Provides detailed project description and disclaimer |                              |                               | ## **Community Node Disclaimer** ... (full extended content describing project context)       |
| Sticky Note3                   | Sticky Note             | Labels block 1.3 Data Storage and Matching      |                              |                               | ## 3. Save data                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Listen to new Instagram comments):**  
   - Type: Webhook  
   - Set HTTP Method to POST.  
   - Configure a unique webhook path (e.g., `7dc782c9-e042-4b43-9b59-f0b7705d8cc8`).  
   - Set Response Mode to “Respond with Node” to allow custom responses later.  
   - This node will receive Instagram comment events and webhook verification requests.

2. **Add Switch Node (Check first validation):**  
   - Type: Switch  
   - Configure two outputs:  
     - First output “Initial Webhook verification”: Condition checks if `{{$json.query['hub.verify_token']}}` equals `"KlickTipp"` (case-sensitive).  
     - Second output “Processing branch after setup”: Default branch for regular events.  
   - Connect Webhook node output to this Switch node.

3. **Add Respond to Webhook Node (Challenge to validate):**  
   - Type: Respond to Webhook  
   - In Response Body, set expression: `{{$json.query['hub.challenge']}}`  
   - Connect first output of Switch ("Initial Webhook verification") to this node.  
   - This node responds to Meta to confirm webhook subscription.

4. **Add If Node (Does comment contain keyword?):**  
   - Type: If  
   - Condition: Check if the Instagram comment text contains the keyword `"keyword"` (case-sensitive).  
   - Use expression: `{{$json.body.entry[0].changes[0].value.text}}` contains `"keyword"`.  
   - Connect second output of Switch ("Processing branch after setup") to this node.

5. **Add HTTP Request Node (DM form to user):**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://graph.instagram.com/v21.0/me/messages`  
   - Authentication: HTTP Header Auth with Instagram OAuth or Bearer token credentials.  
   - Body Parameters:  
     - `message`: JSON string with personalized text including username (from `{{$json.body.entry[0].changes[0].value.from.username}}`) and link to the form (e.g., JotForm URL with username as a parameter).  
     - `recipient`: JSON with recipient ID from `{{$json.body.entry[0].changes[0].value.from.id}}`.  
   - Connect “true” output of keyword If node to this HTTP Request node.

6. **Add Google Sheets Node (Look for entry in matching table):**  
   - Type: Google Sheets  
   - Operation: Lookup rows with filter  
   - Filter: Match “Instagram username” column with username from webhook payload.  
   - Document ID: Specify Google Sheet URL or ID where the matching table resides.  
   - Sheet Name: Use the sheet with Instagram usernames (usually first sheet).  
   - Credentials: Use OAuth2 credentials for Google Sheets.  
   - Connect HTTP Request node output to this node.

7. **Add If Node (Does entry exist?):**  
   - Type: If  
   - Condition: Check if the lookup result has a `row_number` field (meaning user exists).  
   - Connect Google Sheets lookup output to this node.

8. **Add Google Sheets Node (Add entry to matching table):**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Columns to add:  
     - "Instagram username": from webhook payload.  
     - "Instagram ID comment payload": from webhook payload.  
   - Same Google Sheet and credentials as lookup node.  
   - Connect “false” output (entry does not exist) of previous If node to this node.

9. **Add Sticky Notes (Optional, for clarity):**  
   - Add sticky notes labeling the blocks: Data Reception, Validate and Send Message, Save Data.  
   - Optionally add a large sticky note describing the project overview, requirements, and customization tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses KlickTipp community nodes, which are available for self-hosted n8n instances only.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Workflow context and execution environment requirement.                                                                                                                                                                                                                                            |
| To set up, connect Meta (Instagram) Business account, Google Sheets OAuth, and KlickTipp API credentials. Configure your Meta webhook with this workflow’s webhook URL and verify token “KlickTipp”.                                                                                                                                                                                                                                                                                                                                                                                             | Setup instructions in sticky note content.                                                                                                                                                                                                                                                        |
| Customize the DM message content and form link to suit your brand tone and campaign. Use keywords in the filtering node to trigger different DMs or campaigns.                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Customization tips from sticky note.                                                                                                                                                                                                                                                               |
| Requirements include Meta Instagram Business account, Facebook Graph API with `pages_messaging` permission, Google Sheets OAuth connection, and KlickTipp account with API access.                                                                                                                                                                                                                                                                                                                                                                                                            | Technical prerequisites.                                                                                                                                                                                                                                                                            |
| Project description and rationale: Automates lead capture from Instagram comments by sending personalized DMs with forms, storing leads in Google Sheets, and syncing contacts in KlickTipp for marketing automation.                                                                                                                                                                                                                                                                                                                                                                              | Sticky note detailed content.                                                                                                                                                                                                                                                                       |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation platform. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.