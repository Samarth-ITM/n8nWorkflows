Sync Website Visitors from RB2B to Attio CRM via Slack with Deal Creation

https://n8nworkflows.xyz/workflows/sync-website-visitors-from-rb2b-to-attio-crm-via-slack-with-deal-creation-6973


# Sync Website Visitors from RB2B to Attio CRM via Slack with Deal Creation

### 1. Workflow Overview

This n8n workflow automates the synchronization of website visitor data captured by RB2B into the Attio CRM system, leveraging Slack as the notification channel. It is designed for sales teams seeking to convert anonymous website visitors into actionable leads by automating contact creation and deal management within Attio CRM.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures new visitor notifications sent by RB2B into a specific Slack channel.
- **1.2 Data Parsing and Cleanup:** Extracts and sanitizes visitor details from Slack messages, including name, email, company, LinkedIn profile, and URLs.
- **1.3 CRM Person Lookup:** Queries Attio CRM to check if the visitor already exists based on their email address.
- **1.4 Deal Management Logic:** Decides whether to update existing deals or create new deals associated with contacts.
- **1.5 CRM Record Creation/Update:** Creates new person records or deals in Attio CRM or updates deal stages for existing contacts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new messages in a designated Slack channel where RB2B posts website visitor notifications. It acts as the entry point for the workflow.

- **Nodes Involved:**  
  - `RB2B New Message`

- **Node Details:**

  - **Node Name:** RB2B New Message  
  - **Type:** Slack Trigger  
  - **Technical Role:** Monitors Slack channel events, specifically new messages in a configured channel, triggering the workflow on each new visitor notification.  
  - **Configuration:**  
    - Trigger on `"message"` events only.  
    - Monitors Slack channel with ID `C0712SMCKRN` (named "website-visitors").  
    - Uses Slack bot credentials with required permissions to read channel messages and events.  
  - **Expressions/Variables:** None explicitly configured; uses Slack channel ID and event type filters.  
  - **Input/Output:** No input; outputs Slack message JSON.  
  - **Version Requirements:** Compatible with n8n Slack Trigger node v1.  
  - **Edge Cases/Potential Failures:**  
    - Missing or invalid Slack bot credentials will cause the trigger to fail.  
    - Channel ID must be correct and accessible by the bot.  
    - Slack API rate limits or downtime may delay or block triggering.  
  - **Sub-Workflow:** None.

---

#### 2.2 Data Parsing and Cleanup

- **Overview:**  
  This block parses the raw Slack message content to extract structured visitor details such as full name, first and last names, LinkedIn URL, job title, company name, email, location, and associated URLs. It normalizes and cleans data for downstream processing.

- **Nodes Involved:**  
  - `Cleanup Data`

- **Node Details:**

  - **Node Name:** Cleanup Data  
  - **Type:** Code (JavaScript)  
  - **Technical Role:** Processes Slack message JSON to extract and normalize visitor data fields, splits full name, extracts URLs and domains, handles Slack-specific link formatting.  
  - **Configuration:**  
    - Custom JavaScript code runs on incoming Slack message JSON (`$input.first().json`).  
    - Uses regex and structured traversal of Slack message blocks to extract LinkedIn URLs and other links.  
    - Splits full name into first and last names.  
    - Extracts email via multiple regex patterns, including fallback to general text search.  
    - Extracts domains from URLs excluding LinkedIn.  
  - **Expressions/Variables:**  
    - Uses `$input.first().json` to access Slack message.  
    - Returns JSON with keys: `full_name`, `first_name`, `last_name`, `linkedin`, `title`, `company`, `email`, `location`, `urls`, `domains`, and `primary_domain`.  
  - **Input/Output:**  
    - Input: Slack message JSON from `RB2B New Message` node.  
    - Output: Cleaned and structured visitor data JSON.  
  - **Version Requirements:** Node version 2 for code execution.  
  - **Edge Cases/Potential Failures:**  
    - Slack message structure variations may cause parsing errors or missing data.  
    - Malformed Slack rich text blocks or links may cause extraction failures.  
    - Missing expected fields in message text may result in empty or incomplete data.  
  - **Sub-Workflow:** None.

---

#### 2.3 CRM Person Lookup

- **Overview:**  
  This block queries Attio CRM to determine if the visitor already exists as a contact based on their email address, to avoid duplicate records.

- **Nodes Involved:**  
  - `Get Person`  
  - `Person Exists?` (IF node)

- **Node Details:**

  - **Node Name:** Get Person  
  - **Type:** HTTP Request  
  - **Technical Role:** Sends a POST request to Attio API to search for existing people records filtered by email address extracted from visitor data.  
  - **Configuration:**  
    - Endpoint: `https://api.attio.com/v2/objects/people/records/query`  
    - Method: POST  
    - Body JSON filters email addresses containing `{{ $('Cleanup Data').first().json.email }}`  
    - Uses generic HTTP authentication with configured Attio API credentials.  
  - **Expressions/Variables:** Uses email value from `Cleanup Data` node.  
  - **Input/Output:**  
    - Input: Cleaned visitor data JSON.  
    - Output: JSON response from Attio with zero or one matching person records.  
  - **Version Requirements:** HTTP Request node v4.2 or higher for JSON body and authentication handling.  
  - **Edge Cases/Potential Failures:**  
    - Invalid or expired Attio credentials cause auth failures.  
    - Network issues or API rate limits can cause request failures.  
    - Email field missing or malformed leads to ineffective queries.  
  - **Sub-Workflow:** None.

  - **Node Name:** Person Exists?  
  - **Type:** If node  
  - **Technical Role:** Checks whether the `Get Person` node returned any existing contact records.  
  - **Configuration:**  
    - Condition: Boolean expression checking if `$json.data` exists and has length > 0.  
    - True branch indicates person exists; False branch indicates new contact.  
  - **Expressions/Variables:** Uses `$json.data && $json.data.length > 0` from `Get Person` output.  
  - **Input/Output:**  
    - Input: JSON from `Get Person` node.  
    - Output: Routes workflow to different paths based on contact existence.  
  - **Version Requirements:** If node v2.2 for complex boolean condition.  
  - **Edge Cases/Potential Failures:**  
    - Malformed or empty API response could cause false negatives or errors.  
  - **Sub-Workflow:** None.

---

#### 2.4 Deal Management Logic

- **Overview:**  
  This block decides how to handle deals associated with the contact based on their existing deal status. It uses a switch node to branch logic for contacts with or without existing deals.

- **Nodes Involved:**  
  - `Switch`  
  - `Update Deal Stage`  
  - `Create Associated Deal`  
  - `Create Person`  
  - `Create Deal`

- **Node Details:**

  - **Node Name:** Switch  
  - **Type:** Switch node  
  - **Technical Role:** Routes the workflow depending on whether the existing contact has any associated deals.  
  - **Configuration:**  
    - Two outputs:  
      - "Already has associated deal" when `$json.data[0].values.associated_deals.length > 0`  
      - "No associated deal" when `$json.data[0].values.associated_deals.length === 0`  
  - **Expressions/Variables:** Uses `$json.data[0].values.associated_deals` array length.  
  - **Input/Output:**  
    - Input: Output from `Person Exists?` node (true branch).  
    - Output: Routes to update deal or create new deal accordingly.  
  - **Version Requirements:** Switch node v3.2 supports complex expression evaluation.  
  - **Edge Cases/Potential Failures:**  
    - Missing or malformed `associated_deals` array could cause routing errors.  
  - **Sub-Workflow:** None.

  - **Node Name:** Update Deal Stage  
  - **Type:** HTTP Request  
  - **Technical Role:** Updates the stage of the first associated deal for existing contacts to "RB2B Website Visitor".  
  - **Configuration:**  
    - PATCH request to `https://api.attio.com/v2/objects/deals/records/{{ deal_id }}` where `deal_id` is dynamically extracted from the associated deal record.  
    - Body updates `"stage": "RB2B Website Visitor"`.  
    - Uses HTTP Header Auth credentials for Attio API.  
  - **Expressions/Variables:** Uses expression to extract associated deal target record ID from `Get Person` node output.  
  - **Input/Output:**  
    - Input: Routed from Switch node "Already has associated deal".  
    - Output: Ends workflow or further triggers (not shown).  
  - **Version Requirements:** HTTP Request node v4.2+.  
  - **Edge Cases/Potential Failures:**  
    - If deal ID is missing or invalid, request fails.  
    - Auth or API errors can cause update to fail.  
  - **Sub-Workflow:** None.

  - **Node Name:** Create Associated Deal  
  - **Type:** HTTP Request  
  - **Technical Role:** Creates a new deal associated with an existing person who has no deals yet.  
  - **Configuration:**  
    - POST request to create deal with name "Deal with [Full Name]", stage "RB2B Website Visitor".  
    - Owner email hardcoded as `"yourname@emaildomain.com"` (should be customized).  
    - Associates deal with existing person using their record ID.  
    - Uses HTTP Header Auth credentials.  
  - **Expressions/Variables:** Extracts full name and record ID from `Get Person` node output.  
  - **Input/Output:**  
    - Input: Routed from Switch node "No associated deal".  
    - Output: Ends workflow or further triggers (not shown).  
  - **Version Requirements:** HTTP Request node v4.2+.  
  - **Edge Cases/Potential Failures:**  
    - Missing or invalid person record ID causes failure.  
    - Auth or API errors possible.  
  - **Sub-Workflow:** None.

  - **Node Name:** Create Person  
  - **Type:** HTTP Request  
  - **Technical Role:** Creates a new person record in Attio CRM when the visitor does not already exist.  
  - **Configuration:**  
    - POST request to create person with first name, last name, full name, and email from cleaned visitor data.  
    - Uses generic HTTP authentication with Attio credentials.  
  - **Expressions/Variables:** Uses data from `Cleanup Data` node for person details.  
  - **Input/Output:**  
    - Input: Routed from `Person Exists?` node (false branch).  
    - Output: On success, triggers `Create Deal` node to create associated deal.  
  - **Version Requirements:** HTTP Request node v4.2+.  
  - **Edge Cases/Potential Failures:**  
    - Duplicate email errors if email exists but not found in prior query (race condition).  
    - Missing mandatory fields may cause API rejection.  
    - Auth or API errors possible.  
  - **Sub-Workflow:** None.

  - **Node Name:** Create Deal  
  - **Type:** HTTP Request  
  - **Technical Role:** Creates a new deal associated with the newly created person.  
  - **Configuration:**  
    - POST request to create deal with name "Deal with [Full Name]", stage "RB2B Website Visitor", value 0, owner email hardcoded.  
    - Associates deal with newly created person via returned record ID.  
    - Uses HTTP Header Auth credentials.  
  - **Expressions/Variables:** Extracts full name and person record ID from `Create Person` response.  
  - **Input/Output:**  
    - Input: Triggered after `Create Person`.  
    - Output: Ends workflow or further triggers (not shown).  
  - **Version Requirements:** HTTP Request node v4.2+.  
  - **Edge Cases/Potential Failures:**  
    - Errors if person record ID missing or invalid.  
    - Auth or API errors.  
  - **Sub-Workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                                     | Input Node(s)     | Output Node(s)           | Sticky Note                                                                                                          |
|---------------------|-------------------|----------------------------------------------------|-------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| RB2B New Message    | Slack Trigger     | Receives new website visitor notifications from RB2B Slack channel | None              | Cleanup Data             | ## 1. RB2B Slack Trigger [Read more about the Slack Trigger node](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.slacktrigger) Monitors your designated Slack channel for new website visitor notifications from RB2B. Each notification contains structured visitor data including contact details and website activity. |
| Cleanup Data        | Code              | Parses and cleans visitor data from Slack message  | RB2B New Message  | Get Person               | ## 2. Parse RB2B Visitor Data [Read more about the Code node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code) Extracts and structures visitor information from RB2B Slack messages. Handles name splitting, URL extraction, domain parsing, and data cleaning to prepare for CRM import. |
| Get Person          | HTTP Request      | Queries Attio CRM to find existing contact by email | Cleanup Data      | Person Exists?           | ## 3. Check if Person Exists in Attio [Read more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) Searches Attio CRM using the visitor's email address to prevent duplicate contact creation. This ensures data integrity and proper lead management. |
| Person Exists?      | If                | Checks if contact exists in Attio                   | Get Person        | Switch (true), Create Person (false) |                                                                                                                      |
| Switch              | Switch            | Branches deal management logic based on existing deals | Person Exists?    | Update Deal Stage, Create Associated Deal | ## 4. Intelligent Deal Management [Read more about the Switch node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch) Smart logic that handles deal creation differently based on contact status: - New contacts: Creates person record first, then associated deal - Existing contacts: Checks for existing deals and either updates or creates new ones |
| Update Deal Stage   | HTTP Request      | Updates the deal stage for existing deals           | Switch            | None                     |                                                                                                                      |
| Create Associated Deal | HTTP Request    | Creates a new deal associated with an existing contact without deals | Switch            | None                     |                                                                                                                      |
| Create Person       | HTTP Request      | Creates new contact in Attio if not existing        | Person Exists? (false) | Create Deal              |                                                                                                                      |
| Create Deal         | HTTP Request      | Creates new deal associated with newly created contact | Create Person     | None                     | ## 5. Create/Update Attio Records [Read more about Attio API](https://developers.attio.com/) Final step that creates new contact records and deals in Attio CRM. All RB2B leads are tagged with source information and assigned appropriate deal stages for sales follow-up. |
| Sticky Note1        | Sticky Note       | Overview and purpose of workflow                     | None              | None                     | ## 🎯 Sync RB2B Website Visitors to Attio CRM with Deal Creation … Happy Lead Converting!                              |
| Sticky Note         | Sticky Note       | Explanation of Slack Trigger node                    | None              | None                     | ## 1. RB2B Slack Trigger …                                                                                             |
| Sticky Note2        | Sticky Note       | Explanation of Code node for data parsing            | None              | None                     | ## 2. Parse RB2B Visitor Data …                                                                                        |
| Sticky Note3        | Sticky Note       | Explanation of HTTP Request node for Attio person lookup | None              | None                     | ## 3. Check if Person Exists in Attio …                                                                               |
| Sticky Note4        | Sticky Note       | Explanation of deal management logic                  | None              | None                     | ## 4. Intelligent Deal Management …                                                                                    |
| Sticky Note5        | Sticky Note       | Explanation of final creation/update steps in Attio  | None              | None                     | ## 5. Create/Update Attio Records …                                                                                    |
| Sticky Note6        | Sticky Note       | Setup instructions and prerequisites                  | None              | None                     | ### Setup Required! Before activating this workflow: 1. Configure RB2B → Slack integration 2. Add Slack bot credentials to trigger node 3. Add Attio API credentials to all HTTP nodes 4. Select correct Slack channel in trigger 5. Test with sample RB2B notification Missing credentials will cause workflow failures! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Name: `RB2B New Message`  
   - Type: Slack Trigger  
   - Configure to trigger on new `"message"` events.  
   - Set Channel ID to the Slack channel where RB2B posts visitor notifications (e.g., `C0712SMCKRN`).  
   - Attach Slack bot credentials with read message and event permissions.  
   - This node acts as the workflow entry point.

2. **Add Code Node for Data Parsing**  
   - Name: `Cleanup Data`  
   - Type: Code (JavaScript)  
   - Connect input from `RB2B New Message`.  
   - Paste JavaScript code that:  
     - Extracts visitor fields from Slack message text and blocks.  
     - Splits full name into first and last names.  
     - Extracts LinkedIn URLs, emails, company, title, location.  
     - Extracts all URLs and derives domains excluding LinkedIn.  
     - Outputs structured JSON with all extracted fields.  
   - Set node version to 2.

3. **Add HTTP Request Node to Query Attio**  
   - Name: `Get Person`  
   - Type: HTTP Request  
   - Connect input from `Cleanup Data`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.attio.com/v2/objects/people/records/query`  
     - Body (JSON): filter by email containing `{{ $('Cleanup Data').first().json.email }}`  
     - Authentication: Use generic HTTP credentials for Attio API with required headers and tokens.  
   - Set to send JSON body.

4. **Add If Node to Check Person Existence**  
   - Name: `Person Exists?`  
   - Type: If node  
   - Connect input from `Get Person`.  
   - Condition: Check if `$json.data` exists and has length > 0 (boolean expression).  
   - True branch: Person exists.  
   - False branch: Person does not exist.

5. **Create Person for New Visitors**  
   - Name: `Create Person`  
   - Type: HTTP Request  
   - Connect input from `Person Exists?` (false branch).  
   - Configure:  
     - Method: POST  
     - URL: `https://api.attio.com/v2/objects/people/records`  
     - Body (JSON): includes first name, last name, full name, email from `Cleanup Data` outputs.  
     - Authentication: Use Attio API credentials.  

6. **Create Deal for New Person**  
   - Name: `Create Deal`  
   - Type: HTTP Request  
   - Connect input from `Create Person`.  
   - Configure:  
     - Method: POST  
     - URL: `https://api.attio.com/v2/objects/deals/records`  
     - Body (JSON): Create deal named "Deal with [Full Name]", stage "RB2B Website Visitor", value 0, owner email (replace with your email), associate with newly created person by record ID from `Create Person`.  
     - Authentication: Use Attio API credentials.

7. **Add Switch Node for Deal Management**  
   - Name: `Switch`  
   - Type: Switch node  
   - Connect input from `Person Exists?` (true branch).  
   - Configure two outputs:  
     - "Already has associated deal" when `associated_deals.length > 0`.  
     - "No associated deal" when `associated_deals.length === 0`.

8. **Update Deal Stage for Existing Deals**  
   - Name: `Update Deal Stage`  
   - Type: HTTP Request  
   - Connect input from `Switch` output "Already has associated deal".  
   - Configure:  
     - Method: PATCH  
     - URL: `https://api.attio.com/v2/objects/deals/records/{{ deal_id }}`, where `deal_id` is the first associated deal's record ID from `Get Person` output.  
     - Body (JSON): Update deal `"stage": "RB2B Website Visitor"`.  
     - Authentication: Attio API credentials.

9. **Create Associated Deal for Existing Contact Without Deals**  
   - Name: `Create Associated Deal`  
   - Type: HTTP Request  
   - Connect input from `Switch` output "No associated deal".  
   - Configure:  
     - Method: POST  
     - URL: `https://api.attio.com/v2/objects/deals/records`  
     - Body (JSON): Create deal with name "Deal with [Full Name]", stage "RB2B Website Visitor", value 0, owner email (replace accordingly), associate with existing person record ID from `Get Person`.  
     - Authentication: Attio API credentials.

10. **Add Sticky Notes for Documentation**  
    - Add sticky notes corresponding to each major block to explain purpose and provide helpful links, replicating the content from the Sticky Notes in the provided workflow.

11. **Credential Setup**  
    - Configure Slack bot credentials with permissions to read messages and events in the RB2B Slack channel.  
    - Configure Attio API credentials (using either HTTP Header Auth or generic HTTP Auth) with required scopes to query and create/update people and deals.

12. **Final Testing and Activation**  
    - Ensure Slack channel ID is correct and RB2B posts messages as expected.  
    - Test the workflow with sample RB2B visitor notifications.  
    - Verify records are created or updated in Attio CRM accordingly.  
    - Adjust owner emails and deal stage names as per your sales process.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow automatically tags all leads with the source "RB2B Website Visitor" for easy tracking in Attio CRM.                   | Workflow Logic                                                                                           |
| Slack Trigger node documentation: https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.slacktrigger                 | Slack Trigger Node                                                                                       |
| Code node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code                                     | Code Node for Data Parsing                                                                               |
| HTTP Request node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest                      | HTTP Request Node                                                                                        |
| Switch node documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch                                | Switch Node                                                                                             |
| Attio API Reference: https://developers.attio.com/                                                                                   | Attio API                                                                                                |
| Setup prerequisites include proper RB2B → Slack integration, Slack bot credentials, Attio API credentials, and Slack channel selection. | Sticky Note6 Setup Instructions                                                                          |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. All data processed is legal and publicly accessible. No illegal, offensive, or protected content is involved.