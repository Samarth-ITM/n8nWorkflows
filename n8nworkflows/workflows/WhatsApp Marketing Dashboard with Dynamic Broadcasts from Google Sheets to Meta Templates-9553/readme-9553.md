WhatsApp Marketing Dashboard with Dynamic Broadcasts from Google Sheets to Meta Templates

https://n8nworkflows.xyz/workflows/whatsapp-marketing-dashboard-with-dynamic-broadcasts-from-google-sheets-to-meta-templates-9553


# WhatsApp Marketing Dashboard with Dynamic Broadcasts from Google Sheets to Meta Templates

### 1. Workflow Overview

This workflow implements a comprehensive WhatsApp Marketing Dashboard backend with dynamic broadcast capabilities, integrating Google Sheets, Meta WhatsApp Business API templates, and n8n Data Tables. It aims to automate sending personalized WhatsApp broadcast messages to contacts stored in Google Sheets by dynamically building messages based on Meta-approved templates retrieved and stored in n8n Data Tables. The workflow also exposes template data to a frontend via a webhook.

The workflow is logically divided into three main functional blocks:

- **1.1 Broadcast Message Sending Flow**  
  Receives a request to send a broadcast message, fetches template details and contacts, dynamically builds personalized WhatsApp messages, and sends them sequentially with delays to comply with Meta policies.

- **1.2 Template Synchronization Flow**  
  Periodically fetches all approved WhatsApp message templates from the Meta API, clears existing entries in the n8n Data Table, and stores the updated template data for use in messaging and frontend display.

- **1.3 Template Retrieval for Frontend Flow**  
  Provides a webhook endpoint that returns the current list of WhatsApp message templates from the n8n Data Table in JSON format, enabling frontend applications to display template options dynamically.

---

### 2. Block-by-Block Analysis

#### 2.1 Broadcast Message Sending Flow

**Overview:**  
This flow handles sending dynamic WhatsApp broadcast messages to contacts listed in a Google Sheet. It triggers on a POST webhook request specifying the template to use, retrieves the corresponding template structure, fetches contact data, builds the message dynamically, sends the message via Meta API, and respects sending rate limits.

**Nodes Involved:**  
- Trigger: Send Broadcast (Webhook)  
- Get Template Structure from Table (Data Table)  
- Get Contacts from Google Sheet (Google Sheets)  
- Dynamically Build Message Body (Code)  
- Send WhatsApp Message (HTTP Request)  
- Wait 1s Between Messages (Wait)  
- Sticky Note  
- Sticky Note5 (Comment on message rate regulation)

**Node Details:**  

- **Trigger: Send Broadcast**  
  - Type: Webhook  
  - Role: Entry point for broadcast requests; listens for POST requests with JSON payload containing `templateName`.  
  - Key Config: Webhook path set, method POST.  
  - Inputs: External HTTP request (e.g., from frontend or Postman).  
  - Outputs: Passes data downstream to get template structure.  
  - Edge cases: Invalid or missing `templateName` in payload; HTTP errors.  

- **Get Template Structure from Table**  
  - Type: Data Table (get operation)  
  - Role: Retrieves template metadata and components based on the requested template name parsed from the webhook payload.  
  - Configuration: Filters by template_name equals the prefix of `templateName` before " - ".  
  - Inputs: From webhook node.  
  - Outputs: Provides template data for message building.  
  - Edge cases: Template not found; malformed data in Data Table.  

- **Get Contacts from Google Sheet**  
  - Type: Google Sheets (read operation)  
  - Role: Fetches all contacts from a configured Google Sheet to send messages to.  
  - Config: Reads all rows from sheet "gid=0" of a specific document ID.  
  - Inputs: From template structure node.  
  - Outputs: Contact list JSON objects.  
  - Credentials: Google OAuth2 required.  
  - Edge cases: Sheet inaccessible; empty or improperly formatted contact data.  

- **Dynamically Build Message Body**  
  - Type: Code (JavaScript)  
  - Role: Constructs the WhatsApp message body dynamically, merging contact data and template structure to build the final API payload.  
  - Logic Highlights:  
    - Parses template components JSON.  
    - Handles dynamic image headers by inserting URLs from contact data.  
    - Replaces placeholders in body text with contact’s full name.  
    - Processes URL buttons to inject phone numbers dynamically.  
    - Builds final API request object compliant with WhatsApp Business API.  
  - Inputs: JSON from contacts and template structure nodes.  
  - Outputs: JSON with `finalApiRequestBody` for sending.  
  - Edge cases: JSON parse errors; missing expected fields; null values in contact data.  

- **Send WhatsApp Message**  
  - Type: HTTP Request  
  - Role: Sends the constructed message to Meta WhatsApp API.  
  - Config: POST to Meta endpoint with phone number ID; uses header auth for access token.  
  - Inputs: Payload from code node.  
  - Outputs: API response.  
  - Credentials: HTTP Header Auth with permanent WhatsApp access token.  
  - Edge cases: API authentication failure, rate limits, message format errors, network timeouts.  

- **Wait 1s Between Messages**  
  - Type: Wait  
  - Role: Pauses execution 1 second between sending each message to comply with Meta rate limits.  
  - Inputs: From Send Message node.  
  - Outputs: None (end of chain).  
  - Edge cases: Workflow execution delays; large contact lists may increase total runtime.  

- **Sticky Note & Sticky Note5**  
  - Provide explanatory comments about the flow purpose and Meta policy compliance regarding message sending rate.

---

#### 2.2 Template Synchronization Flow

**Overview:**  
This scheduled flow runs daily to synchronize WhatsApp message templates from the Meta API into the n8n Data Table, ensuring the latest approved templates are available for messaging and frontend display.

**Nodes Involved:**  
- Sync Daily (Schedule Trigger)  
- Get Existing Templates from Table (Data Table)  
- Clear Existing Templates (Data Table Delete Rows)  
- Get Templates from Meta (HTTP Request)  
- Split Templates into Items (Code)  
- Save Templates to Data Table (Data Table Upsert)  
- Sticky Note1

**Node Details:**  

- **Sync Daily**  
  - Type: Schedule Trigger  
  - Role: Starts the synchronization once daily automatically.  
  - Config: Interval configured for daily execution.  
  - Outputs: Triggers get existing templates.  

- **Get Existing Templates from Table**  
  - Type: Data Table (get)  
  - Role: Retrieves current templates stored in the Data Table.  
  - Inputs: From schedule trigger.  

- **Clear Existing Templates**  
  - Type: Data Table (deleteRows)  
  - Role: Deletes all rows from the Data Table matching retrieved template IDs to clear old data before refresh.  
  - Inputs: From get existing templates.  
  - Outputs: Triggers fetching from Meta API.  

- **Get Templates from Meta**  
  - Type: HTTP Request  
  - Role: Calls Meta WhatsApp Business API to retrieve all message templates for the specified WABA ID.  
  - Config: GET request to endpoint `/v20.0/{WABA_ID}/message_templates` with query fields `name,language,components,category,status`. Uses HTTP Header Auth for access token.  
  - Inputs: From clearing templates node.  
  - Outputs: JSON list of templates.  
  - Edge cases: API auth failure, API rate limits, network errors, invalid WABA ID.  

- **Split Templates into Items**  
  - Type: Code  
  - Role: Converts the array of templates from Meta API into individual n8n items for downstream processing.  
  - Inputs: Templates array from Meta API response.  
  - Outputs: Item per template.  
  - Edge cases: Empty template list, malformed response.  

- **Save Templates to Data Table**  
  - Type: Data Table (upsert)  
  - Role: Inserts or updates each template into the n8n Data Table with fields: template_name, language_code, components_structure (JSON string), template_id, status, category.  
  - Inputs: Items from split node.  
  - Outputs: None (end of chain).  
  - Edge cases: Data Table write failures, incorrect mapping, schema conflicts.  

- **Sticky Note1**  
  - Describes this flow’s purpose as syncing template details from Meta to n8n Data Table.

---

#### 2.3 Template Retrieval for Frontend Flow

**Overview:**  
Provides a webhook endpoint for frontend applications to retrieve the full list of WhatsApp message templates currently stored in the n8n Data Table, enabling dynamic template selection in user-facing dashboards.

**Nodes Involved:**  
- API: Get Templates for Frontend (Webhook)  
- Get All Templates from Table (Data Table)  
- Return Template List as JSON (Respond to Webhook)  
- Sticky Note2 and Sticky Note6

**Node Details:**  

- **API: Get Templates for Frontend**  
  - Type: Webhook  
  - Role: Entry point for frontend GET requests to fetch templates.  
  - Config: Webhook path set; method defaults to GET. Response mode set to use downstream respond node.  
  - Inputs: External HTTP request.  
  - Outputs: Triggers get all templates node.  
  - Edge cases: Invalid webhook call, no templates in Data Table.  

- **Get All Templates from Table**  
  - Type: Data Table (get all)  
  - Role: Fetches all template records from the Data Table with no filters, returning full dataset.  
  - Inputs: From webhook node.  

- **Return Template List as JSON**  
  - Type: Respond to Webhook  
  - Role: Sends the retrieved template list back as JSON response to the API caller.  
  - Inputs: From Data Table node.  
  - Outputs: HTTP response.  

- **Sticky Note2 and Sticky Note6**  
  - Explain this flow enables frontend template viewing and selection.

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                        | Input Node(s)                        | Output Node(s)                    | Sticky Note                                                                                  |
|------------------------------|---------------------|--------------------------------------------------------|------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Trigger: Send Broadcast       | Webhook             | Entry point to trigger broadcast message sending       | -                                  | Get Template Structure from Table |                                                                                              |
| Get Template Structure from Table | Data Table         | Retrieves template info based on requested template    | Trigger: Send Broadcast             | Get Contacts from Google Sheet    |                                                                                              |
| Get Contacts from Google Sheet| Google Sheets       | Retrieves contact list from Google Sheet                | Get Template Structure from Table   | Dynamically Build Message Body    |                                                                                              |
| Dynamically Build Message Body| Code                | Builds dynamic message body combining template & contact | Get Contacts from Google Sheet      | Send WhatsApp Message             |                                                                                              |
| Send WhatsApp Message         | HTTP Request        | Sends the constructed WhatsApp message                  | Dynamically Build Message Body      | Wait 1s Between Messages          |                                                                                              |
| Wait 1s Between Messages      | Wait                | Waits 1 second between messages for rate limit compliance | Send WhatsApp Message               | None                             | **Regulate number of messages sent per minute according to Meta policies** (Sticky Note5)    |
| Sync Daily                   | Schedule Trigger     | Triggers daily synchronization of templates             | -                                  | Get Existing Templates from Table |                                                                                              |
| Get Existing Templates from Table | Data Table         | Fetches current templates stored                         | Sync Daily                        | Clear Existing Templates          |                                                                                              |
| Clear Existing Templates      | Data Table          | Deletes old template records                              | Get Existing Templates from Table  | Get Templates from Meta           |                                                                                              |
| Get Templates from Meta       | HTTP Request        | Fetches templates from Meta WhatsApp API                 | Clear Existing Templates           | Split Templates into Items        |                                                                                              |
| Split Templates into Items    | Code                | Converts template array into individual items            | Get Templates from Meta            | Save Templates to Data Table      |                                                                                              |
| Save Templates to Data Table  | Data Table          | Inserts or updates templates in Data Table               | Split Templates into Items         | None                             |                                                                                              |
| API: Get Templates for Frontend| Webhook             | Entry point for frontend to get templates list           | -                                  | Get All Templates from Table      |                                                                                              |
| Get All Templates from Table  | Data Table          | Retrieves all templates from Data Table                   | API: Get Templates for Frontend    | Return Template List as JSON      |                                                                                              |
| Return Template List as JSON  | Respond to Webhook  | Returns template list as JSON response                    | Get All Templates from Table       | None                             |                                                                                              |
| Sticky Note                  | Sticky Note         | Label for Flow 1: Send Broadcast Message                  | -                                  | -                                |                                                                                              |
| Sticky Note1                 | Sticky Note         | Label for Flow 2: Sync Template Details                   | -                                  | -                                |                                                                                              |
| Sticky Note2                 | Sticky Note         | Label for Flow 3: Get Meta Templates for Frontend         | -                                  | -                                |                                                                                              |
| Sticky Note3                 | Sticky Note         | Instructions and pre-requisites for the full workflow     | -                                  | -                                | Contains setup instructions, links, and usage notes.                                        |
| Sticky Note4                 | Sticky Note         | Overview and contact info for WhatsApp Broadcast Dashboard| -                                  | -                                | Contains author contact and branding info.                                                  |
| Sticky Note5                 | Sticky Note         | Warning about message sending rate limits                 | -                                  | -                                | **Regulate number of messages sent per minute according to Meta policies**                   |
| Sticky Note6                 | Sticky Note         | Explains flow 3 frontend template selection capability    | -                                  | -                                | **This flow 3 is the reason that you are able to view and select the templates from frontend.** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Trigger: Send Broadcast**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique string (e.g., "56e00faa-448a-4465-ae17-06eee704ed66")  
   - Purpose: Receive broadcast trigger with JSON body containing `templateName`.  

2. **Create Data Table Node: Get Template Structure from Table**  
   - Operation: Get  
   - Data Table: "WhatsApp Templates" (create beforehand)  
   - Filter: `template_name` equals first part of `templateName` from webhook (`={{ $json.body.templateName.split(' - ')[0] }}`)  
   - Connect output of webhook to this node.  

3. **Create Google Sheets Node: Get Contacts from Google Sheet**  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID containing contacts  
   - Sheet Name: e.g., "gid=0" or actual sheet name  
   - Credentials: Google OAuth2 account connected  
   - Connect output of Data Table node to this node.  

4. **Create Code Node: Dynamically Build Message Body**  
   - Language: JavaScript  
   - Script: Implement logic to parse template components, inject contact data into header images, body text, and buttons as per sample code.  
   - Input: JSON from Google Sheets and Data Table nodes  
   - Output: JSON object with `finalApiRequestBody` for WhatsApp API.  
   - Connect output of Google Sheets node to this node.  

5. **Create HTTP Request Node: Send WhatsApp Message**  
   - Method: POST  
   - URL: `https://graph.facebook.com/v20.0/{Your Phone Number ID}/messages` (replace placeholder)  
   - Authentication: HTTP Header Auth with your permanent WhatsApp Access Token  
   - Body Content Type: JSON  
   - Body: Use expression `={{ $json.finalApiRequestBody }}` from code node  
   - Connect output of code node to this node.  

6. **Create Wait Node: Wait 1s Between Messages**  
   - Wait Time: 1 second  
   - Connect output of HTTP Request node to this node.  

7. **Configure the above nodes to handle multiple contacts:**  
   - Ensure the flow is set to process each contact sequentially, respecting wait time to avoid rate limits.  

8. **Create Schedule Trigger Node: Sync Daily**  
   - Schedule: Daily trigger at preferred time  
   - Purpose: Automate template synchronization.  

9. **Create Data Table Node: Get Existing Templates from Table**  
   - Operation: Get all from "WhatsApp Templates" Table  
   - Connect output of Schedule Trigger to this node.  

10. **Create Data Table Node: Clear Existing Templates**  
    - Operation: Delete rows  
    - Filter: Use `id` from existing templates  
    - Connect output of previous node to this node.  

11. **Create HTTP Request Node: Get Templates from Meta**  
    - Method: GET  
    - URL: `https://graph.facebook.com/v20.0/{Your WABA ID}/message_templates` (replace placeholder)  
    - Query Parameter: `fields=name,language,components,category,status`  
    - Authentication: HTTP Header Auth (same as WhatsApp token)  
    - Connect output of Delete Rows node to this node.  

12. **Create Code Node: Split Templates into Items**  
    - JavaScript to split Meta API response array into individual items for upserting.  
    - Connect output of HTTP Request node to this node.  

13. **Create Data Table Node: Save Templates to Data Table**  
    - Operation: Upsert  
    - Data Table: "WhatsApp Templates"  
    - Mapping fields: template_name, language_code, components_structure (stringified JSON), template_id, status, category  
    - Filter on template_name for upsert  
    - Connect output of Code node to this node.  

14. **Create Webhook Node: API: Get Templates for Frontend**  
    - HTTP Method: GET (default)  
    - Path: Unique string (e.g., "127eeea4-2009-4555-83a8-42111ce1de73")  
    - Purpose: Provide API endpoint for frontend template list.  

15. **Create Data Table Node: Get All Templates from Table**  
    - Operation: Get all from "WhatsApp Templates"  
    - Connect output of Webhook node to this node.  

16. **Create Respond to Webhook Node: Return Template List as JSON**  
    - Respond with all incoming items  
    - Connect output of Data Table node to this node.  

17. **Credentials Setup:**  
    - Create HTTP Header Auth credential with your permanent WhatsApp Access Token.  
    - Connect Google OAuth2 credential for Google Sheets access.  

18. **Create n8n Data Table Structure:**  
    - Columns: template_name (string), language_code (string), components_structure (string), template_id (string), status (string), category (string)  

19. **Testing:**  
    - Run the sync flow once manually to populate templates.  
    - Test GET webhook to retrieve templates.  
    - Test POST webhook with JSON body `{ "templateName": "your_template_name - en" }` to send broadcast.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow is a complete backend for a WhatsApp Marketing Dashboard requiring Meta App with WhatsApp, Google Sheet, and n8n Data Table configured.                                                                                 | Sticky Note3                                                                                                 |
| Setup instructions and prerequisite accounts explained with useful video playlists and GitHub repo for frontend dashboard code.                                                                                                  | Links: [YouTube Playlist](https://www.youtube.com/playlist?list=PLl_MEz33D2N0G_fvCpAuvEcYidC8pGPfb), [GitHub](https://github.com/anirudhaeran/Whatsapp-Dashboard), [Netlify Hosting](https://youtu.be/9srnyNC1e_o?si=nSFDZRks_19p43jc) |
| Credential setup includes creating HTTP Header Auth with permanent WhatsApp Access Token and Google Sheets OAuth2.                                                                                                               | Sticky Note3 and official n8n docs: https://docs.n8n.io/                                                     |
| The workflow respects Meta policies by including wait times between messages and recommends regulating message frequency accordingly.                                                                                           | Sticky Note5                                                                                                |
| Author and support contact details provided for questions and further assistance.                                                                                                                                                 | Contact: anirudh.n.aeran@gmail.com (Sticky Note4)                                                           |
| Frontend template retrieval flow enables dynamic display and selection of message templates on user dashboards.                                                                                                                  | Sticky Note2 and Sticky Note6                                                                                |

---

**Disclaimer:**  
The text provided is derived solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled are legal and public.