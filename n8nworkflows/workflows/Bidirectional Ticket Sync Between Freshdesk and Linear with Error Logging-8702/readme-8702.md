Bidirectional Ticket Sync Between Freshdesk and Linear with Error Logging

https://n8nworkflows.xyz/workflows/bidirectional-ticket-sync-between-freshdesk-and-linear-with-error-logging-8702


# Bidirectional Ticket Sync Between Freshdesk and Linear with Error Logging

### 1. Workflow Overview

This workflow named **"Freshdesk-Linear Bridge"** facilitates **bidirectional synchronization** of tickets between Freshdesk (a customer support platform) and Linear (an issue tracking/project management tool). It ensures that tickets created or updated in Freshdesk are reflected as issues in Linear, and updates made to Linear issues are correspondingly applied back to Freshdesk tickets. The system includes robust error logging and validation to maintain data integrity and traceability.

**Target Use Cases:**
- Automatically create Linear issues from new or updated Freshdesk tickets.
- Synchronize status and priority changes between platforms using mapped values.
- Update Freshdesk tickets when corresponding Linear issues are modified.
- Maintain clear linkage between Freshdesk tickets and Linear issues for seamless two-way sync.
- Log success and error events for monitoring and troubleshooting.

**Logical Blocks:**

- **1.1 Webhook Triggers & Data Entry:** Entry points capturing Freshdesk and Linear events to initiate synchronization.
- **1.2 Data Transformation & Field Mapping:** Conversion of ticket/issue data between Freshdesk and Linear formats, including priority and status mappings.
- **1.3 API Operations & External Integration:** Calls to Linear and Freshdesk APIs for issue creation and ticket updates, including validation of responses.
- **1.4 Logging & Operation Management:** Recording success and error events with detailed context for both sync directions.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Triggers & Data Entry

**Overview:**  
This block receives incoming webhooks from Freshdesk (for new and updated tickets) and Linear (for updated issues) to trigger the sync workflow. It acts as the entry point and routes incoming data to the appropriate processing steps.

**Nodes Involved:**  
- 🆕 New Ticket Webhook  
- 📄 Update Ticket Webhook  
- 🎣 Linear Issue Updated Webhook  
- Webhook Triggers Note (Sticky Note)  
- Webhook Trigger Note (Sticky Note)

**Node Details:**

- **🆕 New Ticket Webhook**  
  - Type: Webhook  
  - Role: Listens for POST requests at `/create-ticket` path representing new Freshdesk tickets.  
  - Config: HTTP POST, no authentication by default, expects Freshdesk ticket JSON payload.  
  - Input: External Freshdesk webhook call.  
  - Output: Passes data to field mapping function node.  
  - Failure: Missing or malformed payload; webhook endpoint not reachable.

- **📄 Update Ticket Webhook**  
  - Type: Webhook  
  - Role: Listens for POST requests at `/update-ticket` path representing updated Freshdesk tickets.  
  - Config: HTTP POST, similar to above.  
  - Input: External Freshdesk webhook call.  
  - Output: Passes data to field mapping node.  
  - Failure: Same as above.

- **🎣 Linear Issue Updated Webhook**  
  - Type: Webhook  
  - Role: Receives webhook calls from Linear when an issue is updated, triggering reverse sync.  
  - Config: HTTP POST, listens at `/linear-issue-updated`.  
  - Input: External Linear webhook call.  
  - Output: Routes data to Linear-to-Freshdesk field mapping.  
  - Failure: Missing issue data or misconfigured webhook.

- **Webhook Triggers Note**  
  - Type: Sticky Note  
  - Role: Documents the Freshdesk webhook entry points for new and updated tickets.  
  - Content: Describes the role of these webhooks as primary triggers initiating synchronization.

- **Webhook Trigger Note**  
  - Type: Sticky Note  
  - Role: Documents the Linear webhook entry point for issue updates.  
  - Content: Explains this webhook triggers reverse synchronization from Linear to Freshdesk.

---

#### 1.2 Data Transformation & Field Mapping

**Overview:**  
Processes incoming ticket/issue data to transform and map fields appropriately between Freshdesk and Linear formats. This includes converting priority and status values and preparing titles and descriptions for API compatibility.

**Nodes Involved:**  
- 🗺️ Map Freshdesk Fields to Linear  
- 📄 Map Linear to Freshdesk Fields  
- Data Transformation Note (Sticky Note)  
- Sticky Note (Data Transformation for reverse mapping)

**Node Details:**

- **🗺️ Map Freshdesk Fields to Linear**  
  - Type: Function  
  - Role: Converts Freshdesk ticket fields into Linear issue fields.  
  - Configuration:  
    - Maps Freshdesk priority (1=Low, 2=Medium, 3=High, 4=Urgent) to Linear priority (4=Low to 1=Urgent).  
    - Maps Freshdesk status codes (2=Open, 3=Pending, 4=Resolved, 5=Closed) to Linear states ('todo', 'in_progress', 'done', 'canceled').  
    - Sets Linear title from Freshdesk subject or ticket ID fallback.  
    - Sets Linear description from Freshdesk description or placeholder.  
  - Inputs: JSON payload from Freshdesk webhook nodes.  
  - Outputs: Transformed item with Linear-compatible fields.  
  - Failure: Missing fields, unexpected priority/status values (defaults applied).

- **📄 Map Linear to Freshdesk Fields**  
  - Type: Function  
  - Role: Converts Linear issue data back into Freshdesk ticket format for updates.  
  - Configuration:  
    - Maps Linear states ('todo', 'in_progress', 'done', 'canceled') to Freshdesk statuses (2=Open, 3=Pending, 4=Resolved, 5=Closed).  
    - Maps Linear priority (1=Urgent to 4=Low) back to Freshdesk priority (4=Urgent to 1=Low).  
    - Extracts Freshdesk Ticket ID from Linear issue description using regex.  
    - Passes along title and description for update.  
  - Inputs: JSON payload from Linear webhook.  
  - Outputs: JSON with Freshdesk ticket fields ready for update.  
  - Failure: Missing or malformed issue description causing failure to extract ticket ID.

- **Data Transformation Note**  
  - Type: Sticky Note  
  - Role: Explains the importance of field mapping from Freshdesk to Linear to maintain data consistency and format expectations.

- **Sticky Note (Data Transformation for reverse mapping)**  
  - Type: Sticky Note  
  - Role: Details the reverse mapping from Linear back to Freshdesk, highlighting state and priority conversions and ticket ID extraction.

---

#### 1.3 API Operations & External Integration

**Overview:**  
Handles communication with external APIs (Linear and Freshdesk) to create issues, update tickets, and validate responses to ensure reliable synchronization.

**Nodes Involved:**  
- 🎯 Create Linear Issue  
- ✅ Check Linear Creation Success  
- 🔗 Link Freshdesk with Linear ID  
- 🔍 Check if Freshdesk Ticket ID Exists  
- 🎫 Update Freshdesk Ticket  
- API Operations Note  
- API Operations Note1 (Sticky Notes)

**Node Details:**

- **🎯 Create Linear Issue**  
  - Type: HTTP Request  
  - Role: Sends GraphQL mutation to Linear API to create a new issue based on mapped data.  
  - Configuration:  
    - URL: `https://api.linear.app/graphql`  
    - Headers: Content-Type JSON, Authorization with Linear API key from environment variable `$vars.LINEAR_API_KEY`  
    - Body: GraphQL mutation with issue data (priority, state, title, description).  
    - Authentication: HTTP Header (bearer token).  
  - Input: Data from mapping function node.  
  - Output: API response containing creation success and issue details.  
  - Failure: API errors, invalid auth, network issues.

- **✅ Check Linear Creation Success**  
  - Type: If Node  
  - Role: Checks if the Linear API response indicates success (`data.issueCreate.success === true`).  
  - Configuration: Boolean condition on response JSON.  
  - Input: Output of Create Linear Issue node.  
  - Output:  
    - Success path: Proceeds to link Freshdesk ticket with Linear ID.  
    - Failure path: Routes to error logging node.  
  - Failure: Missing or malformed response.

- **🔗 Link Freshdesk with Linear ID**  
  - Type: HTTP Request  
  - Role: Updates the original Freshdesk ticket to include a reference to the newly created Linear issue ID, enabling bidirectional linkage.  
  - Configuration:  
    - URL: Uses Freshdesk domain environment variable `$vars.FRESHDESK_DOMAIN` and ticket ID from mapping node.  
    - Method: PUT or PATCH (implied by update).  
    - Headers: Content-Type JSON, Basic Auth using stored Freshdesk credentials.  
    - Body: JSON payload linking Linear issue ID.  
  - Input: Success output from Linear creation check node.  
  - Output: Passes to success logging node.  
  - Failure: Auth errors, API errors, invalid ticket ID.

- **🔍 Check if Freshdesk Ticket ID Exists**  
  - Type: If Node  
  - Role: Validates that the mapped Linear issue contains a valid Freshdesk ticket ID before attempting update.  
  - Configuration: Checks non-empty `freshdeskTicketId` field.  
  - Input: Mapped Linear data from webhook.  
  - Output:  
    - Exists path: Proceeds to update Freshdesk ticket.  
    - Missing path: Routes to error logging node.  
  - Failure: Null or missing IDs cause fallback error handling.

- **🎫 Update Freshdesk Ticket**  
  - Type: HTTP Request  
  - Role: Sends update request to Freshdesk API to synchronize changes from Linear issue back to the corresponding Freshdesk ticket.  
  - Configuration:  
    - URL: Freshdesk domain + ticket ID from node data.  
    - Method: PUT or PATCH.  
    - Headers: Content-Type JSON, Basic Auth credentials.  
    - Body: JSON payload with updated fields (status, priority, description).  
  - Input: Validated mapped data from If node.  
  - Output: Success logging node.  
  - Failure: API errors, auth failures, invalid ticket IDs.

- **API Operations Note** & **API Operations Note1**  
  - Type: Sticky Notes  
  - Role: Document API integration logic and validation steps for both directions (Freshdesk → Linear and Linear → Freshdesk).

---

#### 1.4 Logging & Operation Management

**Overview:**  
Provides detailed logging of success and failure events during synchronization for both directions. These logs facilitate monitoring, auditing, and troubleshooting.

**Nodes Involved:**  
- ❌ Log Linear Creation Error  
- 🎉 Log Linear Creation Success  
- ⚠️ Log Missing Ticket ID Error  
- ✅ Log Freshdesk Update Success  
- Logging Management Note  
- Logging Error Note

**Node Details:**

- **❌ Log Linear Creation Error**  
  - Type: Function  
  - Role: Logs failure details when Linear issue creation fails, including timestamp, workflow name, Freshdesk ticket ID, and API response.  
  - Outputs an error JSON object for potential downstream handling or storage.  
  - Failure: None (logging only).

- **🎉 Log Linear Creation Success**  
  - Type: Function  
  - Role: Logs success event with detailed info such as Linear issue ID/key, Freshdesk ticket ID, timestamp, and workflow context.  
  - Outputs success JSON object.  
  - Failure: None.

- **⚠️ Log Missing Ticket ID Error**  
  - Type: Function  
  - Role: Logs error when Linear issue lacks a Freshdesk ticket ID, preventing reverse sync.  
  - Includes detailed context for debugging.  
  - Outputs error JSON.  
  - Failure: None.

- **✅ Log Freshdesk Update Success**  
  - Type: Function  
  - Role: Logs successful Freshdesk ticket update with relevant IDs and timestamp.  
  - Outputs success JSON.  
  - Failure: None.

- **Logging Management Note** & **Logging Error Note**  
  - Type: Sticky Notes  
  - Role: Document the logging approach for both forward and reverse synchronization, emphasizing audit trail completeness.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                 | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                     |
|----------------------------------|---------------------|------------------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| 🆕 New Ticket Webhook             | Webhook             | Entry point for new Freshdesk tickets          | External                        | 🗺️ Map Freshdesk Fields to Linear   | 🎣 Webhook Triggers & Data Entry: New Ticket Webhook triggers Freshdesk→Linear sync             |
| 📄 Update Ticket Webhook          | Webhook             | Entry point for updated Freshdesk tickets      | External                        | 🗺️ Map Freshdesk Fields to Linear   | 🎣 Webhook Triggers & Data Entry: Update Ticket Webhook triggers Freshdesk→Linear sync          |
| 🎣 Linear Issue Updated Webhook   | Webhook             | Entry point for updated Linear issues          | External                        | 📄 Map Linear to Freshdesk Fields   | 🎣 Webhook Trigger & Data Entry: Linear webhook triggers Linear→Freshdesk sync                  |
| 🗺️ Map Freshdesk Fields to Linear | Function            | Map Freshdesk ticket data to Linear format     | 🆕 New Ticket Webhook, Update Ticket Webhook | 🎯 Create Linear Issue               | 📄 Data Transformation & Field Mapping: Freshdesk→Linear field mapping                          |
| 🎯 Create Linear Issue            | HTTP Request        | Create Linear issue via GraphQL mutation       | 🗺️ Map Freshdesk Fields to Linear | ✅ Check Linear Creation Success    | 🎯 API Operations & External Integration: Create Linear issue API call                          |
| ✅ Check Linear Creation Success  | If                  | Verify Linear issue creation success            | 🎯 Create Linear Issue            | 🔗 Link Freshdesk with Linear ID, ❌ Log Linear Creation Error | 🎯 API Operations & External Integration: Validate Linear issue creation success                |
| 🔗 Link Freshdesk with Linear ID | HTTP Request        | Update Freshdesk ticket with Linear issue ID  | ✅ Check Linear Creation Success  | 🎉 Log Linear Creation Success      | 🎯 API Operations & External Integration: Link Freshdesk ticket with Linear issue              |
| 🎉 Log Linear Creation Success   | Function            | Log successful Linear issue creation           | 🔗 Link Freshdesk with Linear ID |                                   | 📊 Logging & Operation Management: Log success of Linear issue creation                         |
| ❌ Log Linear Creation Error      | Function            | Log failure in Linear issue creation           | ✅ Check Linear Creation Success (failure path) |                                   | 📊 Logging & Operation Management: Log error on Linear issue creation                           |
| 📄 Map Linear to Freshdesk Fields | Function            | Map Linear issue data to Freshdesk format      | 🎣 Linear Issue Updated Webhook   | 🔍 Check if Freshdesk Ticket ID Exists | 📄 Data Transformation & Field Mapping: Linear→Freshdesk field mapping                         |
| 🔍 Check if Freshdesk Ticket ID Exists | If                  | Check presence of Freshdesk ticket ID          | 📄 Map Linear to Freshdesk Fields | 🎫 Update Freshdesk Ticket, ⚠️ Log Missing Ticket ID Error | 🎯 API Operations & Validation: Validate Freshdesk ticket ID presence                          |
| 🎫 Update Freshdesk Ticket       | HTTP Request        | Update Freshdesk ticket with Linear data       | 🔍 Check if Freshdesk Ticket ID Exists | ✅ Log Freshdesk Update Success     | 🎯 API Operations & Validation: Update Freshdesk ticket via API                                |
| ✅ Log Freshdesk Update Success  | Function            | Log success of Freshdesk ticket update         | 🎫 Update Freshdesk Ticket        |                                   | 📊 Logging & Error Management: Log successful Freshdesk ticket update                          |
| ⚠️ Log Missing Ticket ID Error    | Function            | Log error when no Freshdesk ticket ID found    | 🔍 Check if Freshdesk Ticket ID Exists (failure path) |                                   | 📊 Logging & Error Management: Log missing Freshdesk ticket ID error                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Freshdesk Triggers:**
   - Add a Webhook node named **"🆕 New Ticket Webhook"**  
     - HTTP Method: POST  
     - Path: `create-ticket`  
     - No authentication required (or set as per your security needs).  
   - Add a Webhook node named **"📄 Update Ticket Webhook"**  
     - HTTP Method: POST  
     - Path: `update-ticket`  

2. **Create Webhook Node for Linear Trigger:**
   - Add a Webhook node named **"🎣 Linear Issue Updated Webhook"**  
     - HTTP Method: POST  
     - Path: `linear-issue-updated`  

3. **Create Function Node to Map Freshdesk to Linear Fields:**
   - Name: **"🗺️ Map Freshdesk Fields to Linear"**  
   - Paste JavaScript code to map Freshdesk priority/status to Linear priority/state, and set title/description.  
   - Connect **"🆕 New Ticket Webhook"** and **"📄 Update Ticket Webhook"** nodes outputs to this node.  

4. **Create HTTP Request Node to Create Linear Issue:**
   - Name: **"🎯 Create Linear Issue"**  
   - Method: POST  
   - URL: `https://api.linear.app/graphql`  
   - Authentication: HTTP Header with bearer token using environment variable `$vars.LINEAR_API_KEY`  
   - Headers: Content-Type `application/json`, Authorization `Bearer $vars.LINEAR_API_KEY`  
   - Body: GraphQL mutation with issue data (priority, state, title, description) from previous node.  
   - Connect output of **"🗺️ Map Freshdesk Fields to Linear"** to this node.  

5. **Create If Node to Check Linear Creation Success:**
   - Name: **"✅ Check Linear Creation Success"**  
   - Condition: `$json.data.issueCreate.success === true`  
   - Connect output of **"🎯 Create Linear Issue"** to this node.  

6. **Create HTTP Request Node to Link Freshdesk Ticket with Linear ID:**
   - Name: **"🔗 Link Freshdesk with Linear ID"**  
   - Method: PUT or PATCH (as per Freshdesk API)  
   - URL: `https://{{ $vars.FRESHDESK_DOMAIN }}.freshdesk.com/api/v2/tickets/{{ $('🗺️ Map Freshdesk Fields to Linear').item.json.id }}`  
   - Authentication: Basic Auth with stored Freshdesk credentials  
   - Headers: Content-Type `application/json`  
   - Body: JSON payload linking Linear issue ID to Freshdesk ticket  
   - Connect **"✅ Check Linear Creation Success"** success output to this node.  

7. **Create Function Node to Log Linear Creation Success:**
   - Name: **"🎉 Log Linear Creation Success"**  
   - JavaScript code to log success details (timestamp, IDs).  
   - Connect output of **"🔗 Link Freshdesk with Linear ID"** to this node.  

8. **Create Function Node to Log Linear Creation Error:**
   - Name: **"❌ Log Linear Creation Error"**  
   - JavaScript code to log error details from failed Linear creation.  
   - Connect failure output of **"✅ Check Linear Creation Success"** to this node.  

9. **Create Function Node to Map Linear to Freshdesk Fields:**
   - Name: **"📄 Map Linear to Freshdesk Fields"**  
   - JavaScript code to map Linear state/priority to Freshdesk status/priority and extract Freshdesk ticket ID from description.  
   - Connect output of **"🎣 Linear Issue Updated Webhook"** to this node.  

10. **Create If Node to Check Freshdesk Ticket ID Exists:**
    - Name: **"🔍 Check if Freshdesk Ticket ID Exists"**  
    - Condition: Check that `freshdeskTicketId` field is not empty.  
    - Connect output of **"📄 Map Linear to Freshdesk Fields"** to this node.  

11. **Create HTTP Request Node to Update Freshdesk Ticket:**
    - Name: **"🎫 Update Freshdesk Ticket"**  
    - Method: PUT or PATCH (per Freshdesk API)  
    - URL: `https://{{ $vars.FRESHDESK_DOMAIN }}.freshdesk.com/api/v2/tickets/{{ $json.freshdeskTicketId }}`  
    - Authentication: Basic Auth with Freshdesk credentials  
    - Headers: Content-Type `application/json`  
    - Body: JSON with mapped Freshdesk fields from previous node  
    - Connect **"🔍 Check if Freshdesk Ticket ID Exists"** true output to this node.  

12. **Create Function Node to Log Freshdesk Update Success:**
    - Name: **"✅ Log Freshdesk Update Success"**  
    - JavaScript code to log update success with timestamp and IDs.  
    - Connect output of **"🎫 Update Freshdesk Ticket"** to this node.  

13. **Create Function Node to Log Missing Ticket ID Error:**
    - Name: **"⚠️ Log Missing Ticket ID Error"**  
    - JavaScript code to log error for missing Freshdesk ticket ID in Linear issue.  
    - Connect false output of **"🔍 Check if Freshdesk Ticket ID Exists"** to this node.  

14. **Create Sticky Notes for Documentation:**
    - Add notes describing webhook triggers, data transformation, API operations, and logging as per the provided descriptions.

15. **Configure Credentials:**
    - Set up Freshdesk Basic Auth credentials for API requests.  
    - Set Linear API key in environment variables or credentials manager as `$vars.LINEAR_API_KEY`.  
    - Set Freshdesk domain in `$vars.FRESHDESK_DOMAIN`.

16. **Validate and Test:**
    - Test Freshdesk webhook triggers with sample ticket data.  
    - Verify Linear issue creation and linkage.  
    - Test Linear webhook trigger and Freshdesk ticket update.  
    - Monitor logs for success and error messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow enables seamless bidirectional sync between Freshdesk tickets and Linear issues with field mapping.   | Core project goal                                                                                   |
| Priority and status mappings are explicitly defined and reversible to maintain integrity between platforms.    | Important for correct ticket/issue state sync                                                     |
| Logging nodes provide timestamped audit trails and error capture for troubleshooting sync issues.               | Critical for operational monitoring and debugging                                                  |
| Use environment variables for API keys and domain to maintain security and reusability across environments.    | Best practice for credential management                                                           |
| Freshdesk API documentation: https://developers.freshdesk.com/api/                                            | Reference for ticket update API calls                                                             |
| Linear API documentation: https://developers.linear.app/docs/graphql/getting-started/                          | Reference for GraphQL mutations and issue operations                                              |
| Sticky notes within the workflow provide context and explanations for easier maintenance and onboarding.       | Helpful for team knowledge sharing and future modifications                                       |

---

**Disclaimer:**  
The text above is exclusively derived from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.