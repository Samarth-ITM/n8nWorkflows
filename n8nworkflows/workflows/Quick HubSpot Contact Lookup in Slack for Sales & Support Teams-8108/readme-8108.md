Quick HubSpot Contact Lookup in Slack for Sales & Support Teams

https://n8nworkflows.xyz/workflows/quick-hubspot-contact-lookup-in-slack-for-sales---support-teams-8108


# Quick HubSpot Contact Lookup in Slack for Sales & Support Teams

---
### 1. Workflow Overview

This workflow, titled **"Quick HubSpot Contact Lookup in Slack for Sales & Support Teams,"** enables seamless contact information retrieval from HubSpot directly within Slack channels via a slash command. It is designed to empower sales and support teams to quickly look up contact details by providing either an email address or a HubSpot contact ID.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Receives the slash command request from Slack and verifies that a search input is provided.
- **1.2 Input Parsing and Search Type Determination:** Parses the input text to extract the search parameter and decides whether to treat it as an email or contact ID.
- **1.3 HubSpot Contact Retrieval:** Queries HubSpot either by email or by contact ID accordingly.
- **1.4 Contact Formatting:** Formats the retrieved contact information into a Slack-friendly message.
- **1.5 Slack Response Delivery:** Sends the formatted contact card back to the appropriate Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block listens for incoming slash command requests from Slack, ensures the request contains a message text, and halts execution with an error if the message is empty or missing.

**Nodes Involved:**  
- 🌐 Incoming Slash Command (Webhook)  
- 🔍 Check if Message Exists (If)  
- 🛑 Stop – Missing Message (Stop and Error)

**Node Details:**

- **🌐 Incoming Slash Command**  
  - *Type:* Webhook  
  - *Role:* Entry point receiving POST requests from Slack on path `/hubspot-contact-lookup`.  
  - *Configuration:* HTTP POST method, webhook ID set for Slack integration.  
  - *Key Variables:* `$json.body.text` contains the user’s message.  
  - *Connections:* Output to "🔍 Check if Message Exists".  
  - *Edge Cases:* Invalid or missing HTTP POST requests; Slack token verification is external/not shown.  
  - *Notes:* Critical to ensure Slack correctly triggers this webhook.

- **🔍 Check if Message Exists**  
  - *Type:* If  
  - *Role:* Checks if the incoming message text is non-empty.  
  - *Configuration:* Condition tests if `$json.body.text` is not empty (strict, case-sensitive).  
  - *Connections:* True branch leads to "✂️ Parse Search Input"; False branch to "🛑 Stop – Missing Message".  
  - *Edge Cases:* Empty or whitespace-only messages cause workflow to halt early.

- **🛑 Stop – Missing Message**  
  - *Type:* Stop and Error  
  - *Role:* Terminates workflow and returns an error message "No message found".  
  - *Configuration:* Custom error message for user feedback.  
  - *Connections:* None (end node).  
  - *Edge Cases:* Provides graceful termination on invalid input.

---

#### 2.2 Input Parsing and Search Type Determination

**Overview:**  
Parses the user-provided input to clean and classify it as either an email address or a HubSpot contact ID. This classification informs which HubSpot API endpoint to use for the lookup.

**Nodes Involved:**  
- ✂️ Parse Search Input (Code)  
- 🔀 Determine Search Type (If)

**Node Details:**

- **✂️ Parse Search Input**  
  - *Type:* Code  
  - *Role:* Extracts and trims the search input from the incoming Slack message JSON.  
  - *Configuration:* JavaScript code validates input presence, uses regex to detect if input matches email format, else treats it as contact ID.  
  - *Key Expressions:*  
    - Email regex pattern for validation.  
    - Returns: `searchValue`, `searchType` ('email' or 'id'), plus Slack `channelId`, `userId`, `responseUrl`.  
  - *Connections:* Output to "🔀 Determine Search Type".  
  - *Edge Cases:* Throws error if input is empty or invalid format (halts workflow).

- **🔀 Determine Search Type**  
  - *Type:* If  
  - *Role:* Routes workflow based on `searchType` (`email` vs `id`).  
  - *Configuration:* Checks if `$json.searchType === 'email'`.  
  - *Connections:*  
    - True branch to "📧 Search Contact by Email".  
    - False branch to "🆔 Get Contact by ID".  
  - *Edge Cases:* Only two paths supported; unexpected values not handled explicitly.

---

#### 2.3 HubSpot Contact Retrieval

**Overview:**  
Performs the actual HubSpot API calls to retrieve contact details either by email search or direct contact ID lookup.

**Nodes Involved:**  
- 📧 Search Contact by Email (HubSpot)  
- 🆔 Get Contact by ID (HubSpot)

**Node Details:**

- **📧 Search Contact by Email**  
  - *Type:* HubSpot node  
  - *Role:* Searches HubSpot contacts filtering by email address.  
  - *Configuration:*  
    - Operation: Search.  
    - Filter group: filters on property `email` equal to the provided email.  
    - Returns selected properties: firstname, lastname, email, phone.  
    - Auth: HubSpot App Token credentials required.  
  - *Connections:* Output to "📝 Format Contact Info (Email Search)".  
  - *Edge Cases:* No contact found returns empty; API errors (auth, rate limiting) possible.

- **🆔 Get Contact by ID**  
  - *Type:* HubSpot node  
  - *Role:* Retrieves a contact by its unique HubSpot contact ID.  
  - *Configuration:*  
    - Operation: Get.  
    - Contact ID dynamically set from input.  
    - Auth: HubSpot App Token credentials.  
  - *Connections:* Output to "📝 Format Contact Info (ID Search)".  
  - *Edge Cases:* Invalid ID leads to not found or API error.

---

#### 2.4 Contact Formatting

**Overview:**  
Formats the retrieved contact data into a clean, human-readable Slack message with key details.

**Nodes Involved:**  
- 📝 Format Contact Info (Email Search) (Code)  
- 📝 Format Contact Info (ID Search) (Code)

**Node Details:**

- **📝 Format Contact Info (Email Search)**  
  - *Type:* Code  
  - *Role:* Processes HubSpot search by email response to extract contact properties.  
  - *Configuration:* JS code checks presence of `properties`. Extracts firstname, lastname, email, phone, company, deal stage.  
  - *Output:* Slack formatted message string with contact info.  
  - *Connections:* Output to "💬 Send Contact Info to Slack".  
  - *Edge Cases:* Returns 'Contact not found' if data missing; robust to missing fields by defaulting to 'N/A'.

- **📝 Format Contact Info (ID Search)**  
  - *Type:* Code  
  - *Role:* Processes HubSpot get by ID response to extract contact properties.  
  - *Configuration:* JS code handles different response structures (`results` array or `properties` object). Extracts full name, email, phone.  
  - *Output:* Slack formatted message string.  
  - *Connections:* Output to "💬 Send Contact Info to Slack".  
  - *Edge Cases:* Handles empty results gracefully, returns failure message.

---

#### 2.5 Slack Response Delivery

**Overview:**  
Sends the formatted contact information message back to the original Slack channel.

**Node Involved:**  
- 💬 Send Contact Info to Slack (Slack)

**Node Details:**

- **💬 Send Contact Info to Slack**  
  - *Type:* Slack node  
  - *Role:* Posts a message to a specified Slack channel containing contact details.  
  - *Configuration:*  
    - Sends message text from previous node’s output (`$json.message`).  
    - Channel ID set statically (replace `"YOUR_CHANNEL_ID"` with actual target).  
    - Authenticated with Slack API credentials.  
  - *Connections:* None (terminal node).  
  - *Edge Cases:* Slack API errors (auth failures, invalid channel), message formatting issues.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                           |
|-------------------------------|---------------------|------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| 🌐 Incoming Slash Command      | Webhook             | Receives slash command from Slack  | —                           | 🔍 Check if Message Exists   | This section handles receiving incoming requests (e.g., from Slack) and ensuring they contain a valid message. |
| 🔍 Check if Message Exists     | If                  | Validates presence of message text | 🌐 Incoming Slash Command    | ✂️ Parse Search Input, 🛑 Stop – Missing Message | See above                                                                                           |
| 🛑 Stop – Missing Message      | Stop and Error      | Stops workflow on missing message  | 🔍 Check if Message Exists   | —                           | See above                                                                                           |
| ✂️ Parse Search Input          | Code                | Parses and classifies input         | 🔍 Check if Message Exists   | 🔀 Determine Search Type     | Parses the incoming message to extract and clean the search value. Determines if input is email or ID. |
| 🔀 Determine Search Type       | If                  | Routes based on search type         | ✂️ Parse Search Input        | 📧 Search Contact by Email, 🆔 Get Contact by ID | See above                                                                                           |
| 📧 Search Contact by Email     | HubSpot              | Searches contact by email           | 🔀 Determine Search Type     | 📝 Format Contact Info (Email Search) | If input is email → searches HubSpot using email filter.                                            |
| 🆔 Get Contact by ID           | HubSpot              | Retrieves contact by ID             | 🔀 Determine Search Type     | 📝 Format Contact Info (ID Search) | If input is ID → retrieves contact record directly by ID.                                           |
| 📝 Format Contact Info (Email Search) | Code                | Formats contact data for Slack      | 📧 Search Contact by Email   | 💬 Send Contact Info to Slack | Formats retrieved HubSpot contact properties into Slack-friendly message.                           |
| 📝 Format Contact Info (ID Search)    | Code                | Formats contact data for Slack      | 🆔 Get Contact by ID         | 💬 Send Contact Info to Slack | See above                                                                                           |
| 💬 Send Contact Info to Slack  | Slack                | Sends formatted info to channel    | 📝 Format Contact Info (Email Search), 📝 Format Contact Info (ID Search) | —                           | Sends contact information to designated Slack channel.                                              |
| Sticky Note                   | Sticky Note          | Documentation                      | —                           | —                           | Explains Input Reception and Validation section.                                                   |
| Sticky Note1                  | Sticky Note          | Documentation                      | —                           | —                           | Explains Input Parsing and Search Type block.                                                     |
| Sticky Note2                  | Sticky Note          | Documentation                      | —                           | —                           | Explains HubSpot contact retrieval nodes.                                                         |
| Sticky Note3                  | Sticky Note          | Documentation                      | —                           | —                           | Explains contact formatting nodes.                                                                |
| Sticky Note4                  | Sticky Note          | Documentation                      | —                           | —                           | Explains Slack message sending node.                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `🌐 Incoming Slash Command`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `hubspot-contact-lookup`  
   - Purpose: To receive slash command requests from Slack.  
   - No authentication inside node; ensure Slack app configured to send here.

2. **Create If Node to Validate Message:**  
   - Name: `🔍 Check if Message Exists`  
   - Type: If  
   - Condition: Check if expression `{{$json.body.text}}` is not empty (string, not empty, strict).  
   - Connect input from the webhook node.  
   - True → continue; False → stop node.

3. **Create Stop and Error Node for Empty Input:**  
   - Name: `🛑 Stop – Missing Message`  
   - Type: Stop and Error  
   - Error Message: "No message found"  
   - Connect False output from the If node here.

4. **Create Code Node to Parse Input:**  
   - Name: `✂️ Parse Search Input`  
   - Type: Code  
   - JavaScript code:  
     - Extract `$json.body.text`, trim whitespace.  
     - If empty, throw error with message "Please provide an email address or HubSpot contact ID".  
     - Use regex to test if input is an email.  
     - Return JSON with keys: `searchValue`, `searchType` ('email' or 'id'), plus Slack `channelId`, `userId`, `responseUrl`.  
   - Connect True output of validation If node here.

5. **Create If Node to Determine Search Type:**  
   - Name: `🔀 Determine Search Type`  
   - Type: If  
   - Condition: Check if `{{$json.searchType}}` equals `email`.  
   - Connect input from parse input code node.  
   - True → search by email node, False → get by ID node.

6. **Create HubSpot Node to Search by Email:**  
   - Name: `📧 Search Contact by Email`  
   - Type: HubSpot  
   - Operation: Search  
   - Filter: Filter group with filter on property `email` equals `{{$json.searchValue}}`.  
   - Additional Fields: Return properties `firstname`, `lastname`, `email`, `phone`.  
   - Credentials: Use HubSpot API credentials with App Token authentication.  
   - Connect True output of search type If node here.

7. **Create HubSpot Node to Get Contact by ID:**  
   - Name: `🆔 Get Contact by ID`  
   - Type: HubSpot  
   - Operation: Get Contact  
   - Contact ID: `{{$json.searchValue}}` (mode: id)  
   - Credentials: HubSpot API credentials as above.  
   - Connect False output of search type If node here.

8. **Create Code Node to Format Contact Info for Email Search:**  
   - Name: `📝 Format Contact Info (Email Search)`  
   - Type: Code  
   - JavaScript code:  
     - Check response contains `properties`.  
     - Extract `firstname`, `lastname`, `email`, `phone`, `company`, `dealstage`.  
     - Format a multi-line Slack message with these fields, defaulting missing fields to "N/A".  
     - Return JSON with `success: true` and `message` (the formatted string).  
   - Connect output of email search HubSpot node here.

9. **Create Code Node to Format Contact Info for ID Search:**  
   - Name: `📝 Format Contact Info (ID Search)`  
   - Type: Code  
   - JavaScript code:  
     - Handle response structure variants (`results` array or `properties` object).  
     - Extract `firstname`, `lastname`, `hs_full_name_or_email`, `email`, `phone`.  
     - Format a Slack-friendly message string.  
     - Return JSON with `success: true`, `message`, plus Slack `channelId`, `responseUrl`.  
     - If no contact found, return failure message similarly.  
   - Connect output of get by ID HubSpot node here.

10. **Create Slack Node to Send Message:**  
    - Name: `💬 Send Contact Info to Slack`  
    - Type: Slack  
    - Parameters:  
      - Text: `{{$json.message}}` (from format code nodes)  
      - Channel: Set to intended Slack channel ID (replace `"YOUR_CHANNEL_ID"`).  
    - Credentials: Slack API credentials with appropriate scopes to post messages.  
    - Connect outputs of both format contact info code nodes here.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires valid API credentials for both HubSpot (App Token) and Slack (API Token).   | Credential setup must be done in n8n credentials manager.                                      |
| The Slack slash command must be configured to send POST requests to the webhook URL at `/hubspot-contact-lookup`. | Slack App configuration in Slack admin dashboard.                                              |
| Input validation uses regex for email detection: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`                      | Ensures robust determination of email vs. ID.                                                  |
| Error handling gracefully stops workflow if input is missing or contact is not found.               | Prevents unnecessary API calls and informs users promptly.                                     |
| Slack message formatting uses Markdown-like syntax for clarity (bold, emojis).                      | Enhances readability of contact card in Slack channels.                                        |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and maintaining the HubSpot contact lookup workflow integrated with Slack.