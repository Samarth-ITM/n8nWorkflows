Convert Marker.io Visual Bug Reports to Zendesk Support Tickets with Technical Context

https://n8nworkflows.xyz/workflows/convert-marker-io-visual-bug-reports-to-zendesk-support-tickets-with-technical-context-7390


# Convert Marker.io Visual Bug Reports to Zendesk Support Tickets with Technical Context

### 1. Workflow Overview

This workflow automates the conversion of visual bug reports submitted via Marker.io into fully detailed Zendesk support tickets. It targets customer support, product, and development teams who want to streamline bug reporting by automatically translating Marker.io issue data into Zendesk tickets enriched with technical context and metadata.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives Marker.io webhook POST requests containing new issue data.
- **1.2 Data Formatting:** Extracts and formats key issue details and technical context from the Marker.io payload.
- **1.3 User Management:** Creates or updates the reporter as a Zendesk user based on the extracted data.
- **1.4 Ticket Creation:** Opens a new Zendesk ticket using the formatted issue details.
- **1.5 Add Internal Comment:** Appends a detailed internal comment to the ticket with technical metadata, links, and custom data for enhanced troubleshooting.

Supporting the workflow are sticky notes that provide a comprehensive explanation, setup instructions, and troubleshooting guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Receives incoming webhook POST requests from Marker.io when a new issue is created.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - *Type:* Webhook Trigger  
    - *Role:* Entry point for Marker.io issue data  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: Auto-generated unique webhook ID  
      - No authentication, open to Marker.io endpoint calls  
    - *Input/Output:*  
      - Input: External HTTP request  
      - Output: JSON payload of Marker.io issue data  
    - *Edge Cases:*  
      - Incorrect webhook URL in Marker.io will cause no trigger  
      - Payload format changes may break downstream parsing  
      - Network or connectivity issues may delay or fail trigger  

#### 2.2 Data Formatting

- **Overview:**  
Parses the raw Marker.io webhook JSON payload and extracts relevant issue details, including title, description, reporter information, technical context (browser, OS, website URL), and custom data. Formats these into structured strings for ticket and comment bodies.

- **Nodes Involved:**  
  - Format Marker.io Data (Code Node)

- **Node Details:**  
  - **Format Marker.io Data**  
    - *Type:* Code (JavaScript)  
    - *Role:* Data transformation and message formatting  
    - *Configuration:*  
      - Extracts fields such as issue title, description, priority, issue type, reporter email/name/role, URLs, browser, OS, due date, project ID, and context string  
      - Creates two formatted message bodies: one concise for the ticket comment and one detailed for an internal note including technical metadata  
      - Converts dueDate to locale date string if present  
      - Serializes custom data fields into bullet list  
    - *Key Expressions:*  
      - Uses `$input.first().json.body.data` to access Marker.io data  
      - Template literals to build message bodies  
    - *Input/Output:*  
      - Input: Webhook JSON payload  
      - Output: JSON with simplified and formatted issue data fields  
    - *Edge Cases:*  
      - Missing or malformed fields in payload may cause undefined values  
      - Date parsing might fail if dueDate is invalid or null  
      - Custom data may be empty or contain unexpected types  
      - Expression errors if expected properties do not exist  

#### 2.3 User Management

- **Overview:**  
Creates or updates a Zendesk user corresponding to the reporter of the issue, ensuring the user exists before ticket creation.

- **Nodes Involved:**  
  - Create/Update User (HTTP Request)

- **Node Details:**  
  - **Create/Update User**  
    - *Type:* HTTP Request  
    - *Role:* Zendesk API call to create or update user  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://[REPLACE_SUBDOMAIN].zendesk.com/api/v2/users/create_or_update.json`  
      - Body: JSON with user name, email, and role extracted from formatted data  
      - Authentication: Zendesk API token via predefined credentials  
      - Continue On Fail: true (workflow proceeds even if user API call fails)  
    - *Key Expressions:*  
      - Uses expression to build user JSON from formatted data:  
        ```json
        { "name": $json.reporterName, "email": $json.reporterEmail, "role": $json.reporterRole }
        ```  
    - *Input/Output:*  
      - Input: Formatted data node output  
      - Output: Zendesk user creation/update response JSON  
    - *Edge Cases:*  
      - API token with insufficient permissions may cause auth errors  
      - Invalid email format may cause user creation failure  
      - Rate limits or Zendesk service issues could cause failures  
      - Continue On Fail protects workflow from halting here  

#### 2.4 Ticket Creation

- **Overview:**  
Creates a new Zendesk support ticket with the issue title and description, assigns priority, tags, and custom fields based on Marker.io data.

- **Nodes Involved:**  
  - Create Ticket (HTTP Request)

- **Node Details:**  
  - **Create Ticket**  
    - *Type:* HTTP Request  
    - *Role:* Zendesk API call to open a new ticket  
    - *Configuration:*  
      - Method: POST  
      - URL: `https://[REPLACE_SUBDOMAIN].zendesk.com/api/v2/tickets.json`  
      - Body: JSON including:  
        - subject (issue title)  
        - comment body (issue description)  
        - requester email and name  
        - priority (mapped from Marker.io data)  
        - tags including "marker-io" and issue type  
        - custom field with Marker ID (custom field ID placeholder `123456` to be replaced)  
      - Authentication: Zendesk API token via predefined credentials  
    - *Key Expressions:*  
      - Accesses formatted data for all ticket fields, e.g.:  
        - `$('Format Marker.io Data').item.json.messageTitle`  
        - `$('Format Marker.io Data').item.json.priority`  
      - Custom fields array contains one custom field with Marker ID  
    - *Input/Output:*  
      - Input: Output from Create/Update User node  
      - Output: Zendesk ticket creation response JSON (includes ticket ID)  
    - *Edge Cases:*  
      - Missing required fields may cause ticket creation failure  
      - Custom field ID must be set correctly to avoid data loss  
      - API token scope must include ticket creation permissions  
      - Rate limits or network failures could interrupt operation  

#### 2.5 Add Internal Comment

- **Overview:**  
Adds a private/internal comment to the newly created Zendesk ticket containing detailed technical metadata, Marker.io links, browser/OS information, and custom data for developer context.

- **Nodes Involved:**  
  - Add Internal Comment (HTTP Request)

- **Node Details:**  
  - **Add Internal Comment**  
    - *Type:* HTTP Request  
    - *Role:* Zendesk API call to update ticket with internal comment  
    - *Configuration:*  
      - Method: PUT  
      - URL: `https://[REPLACE_SUBDOMAIN].zendesk.com/api/v2/tickets/{{ $json.ticket.id }}.json` (ticket ID from previous node)  
      - Body: JSON with comment body set to detailed note (private: `public: false`)  
      - Authentication: Zendesk API token via predefined credentials  
    - *Key Expressions:*  
      - Uses ticket ID from `Create Ticket` response  
      - Inserts detailed internal note from `Format Marker.io Data` node output  
    - *Input/Output:*  
      - Input: Output from Create Ticket node (contains ticket ID)  
      - Output: Zendesk ticket update confirmation  
    - *Edge Cases:*  
      - Missing or invalid ticket ID will cause failure  
      - API token must have ticket update permissions  
      - Network or API errors may cause comment addition to fail  

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                      | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                                     |
|-----------------------|-------------------|------------------------------------|----------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook               | Webhook           | Receives Marker.io webhook data    | -                    | Format Marker.io Data    | See Sticky Note2 for detailed workflow description and setup instructions                                       |
| Format Marker.io Data  | Code              | Extracts and formats Marker.io data| Webhook              | Create/Update User       | See Sticky Note2 for detailed workflow description and setup instructions                                       |
| Create/Update User     | HTTP Request      | Creates or updates Zendesk user    | Format Marker.io Data | Create Ticket            | See Sticky Note2 for detailed workflow description and setup instructions<br>See Sticky Note3 for troubleshooting|
| Create Ticket         | HTTP Request      | Creates Zendesk ticket             | Create/Update User    | Add Internal Comment     | See Sticky Note2 for detailed workflow description and setup instructions<br>See Sticky Note3 for troubleshooting|
| Add Internal Comment   | HTTP Request      | Adds internal comment to ticket    | Create Ticket        | -                        | See Sticky Note2 for detailed workflow description and setup instructions<br>See Sticky Note3 for troubleshooting|
| Sticky Note2           | Sticky Note       | Workflow overview and instructions | -                    | -                        | # Marker.io to Zendesk Integration<br>Full explanation, setup, benefits, and use cases                         |
| Sticky Note3           | Sticky Note       | Troubleshooting guide               | -                    | -                        | Troubleshooting tips for webhook, user creation, ticket creation, internal comment, and custom field issues    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: (auto-generate or specify unique endpoint)  
   - Purpose: Receive Marker.io issue webhook payloads  

2. **Add Code Node "Format Marker.io Data"**  
   - Type: Code (JavaScript)  
   - Input: Output from Webhook node  
   - Code: Extract `body.data` from webhook JSON payload  
     - Extract fields: title, description, reporter email/name/role, priority, issue type, Marker ID, URLs, browser info, OS info, website URL, context string, due date, project ID, custom data  
   - Format two message bodies:  
     - `messageBody`: Concise issue title + description  
     - `noteBody`: Detailed internal comment with technical context and Marker.io links  
   - Output: JSON object with all extracted fields and message bodies  

3. **Create HTTP Request Node "Create/Update User"**  
   - Method: POST  
   - URL: `https://[YOUR_SUBDOMAIN].zendesk.com/api/v2/users/create_or_update.json`  
   - Authentication: Zendesk API token credentials  
   - Body (JSON):  
     ```json
     {
       "user": {
         "name": {{$json["reporterName"]}},
         "email": {{$json["reporterEmail"]}},
         "role": {{$json["reporterRole"]}}
       }
     }
     ```  
   - Connect input from "Format Marker.io Data"  
   - Set "Continue On Fail" to true to avoid halting workflow on user creation failure  

4. **Create HTTP Request Node "Create Ticket"**  
   - Method: POST  
   - URL: `https://[YOUR_SUBDOMAIN].zendesk.com/api/v2/tickets.json`  
   - Authentication: Zendesk API token credentials  
   - Body (JSON):  
     ```json
     {
       "ticket": {
         "subject": {{$json["messageTitle"]}},
         "comment": { "body": {{$json["messageBody"]}} },
         "requester": {
           "email": {{$json["reporterEmail"]}},
           "name": {{$json["reporterName"]}}
         },
         "priority": {{$json["priority"]}},
         "tags": ["marker-io", {{$json["issueType"]}}],
         "custom_fields": [
           { "id": 123456, "value": {{$json["markerId"]}} }
         ]
       }
     }
     ```  
   - Connect input from "Create/Update User" node  
   - Replace `123456` with your actual Zendesk custom field ID for Marker ID  

5. **Create HTTP Request Node "Add Internal Comment"**  
   - Method: PUT  
   - URL: `https://[YOUR_SUBDOMAIN].zendesk.com/api/v2/tickets/{{ $json.ticket.id }}.json` (use expression to get ticket ID from previous node)  
   - Authentication: Zendesk API token credentials  
   - Body (JSON):  
     ```json
     {
       "ticket": {
         "comment": {
           "body": {{$json["nodeBody"]}},
           "public": false
         }
       }
     }
     ```  
   - Connect input from "Create Ticket" node  

6. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - One sticky note describing the workflow overview, benefits, use cases, setup instructions, and data captured (content from Sticky Note2)  
   - One sticky note with troubleshooting tips for common integration issues (content from Sticky Note3)  

7. **Set up Credentials in n8n**  
   - Create Zendesk API credentials with your Zendesk subdomain and API token  
   - Assign these credentials to all HTTP Request nodes interacting with Zendesk  

8. **Configure Marker.io Webhook**  
   - Copy the webhook URL generated by the Webhook node  
   - In Marker.io workspace settings, create a webhook for "Issue Created" event pointing to this URL  

9. **Test the Integration**  
   - Submit a test bug via Marker.io widget  
   - Confirm user creation/update in Zendesk  
   - Verify ticket creation with correct subject, description, tags, priority, and custom field  
   - Check internal comment for technical details and Marker.io links  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                              | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow creates a seamless integration between Marker.io bug reports and Zendesk tickets, preserving rich technical context for improved support and development collaboration.                                      | Workflow overview and benefits as detailed in Sticky Note2                                                                           |
| Marker.io webhook events documentation for understanding available payload data and webhook setup.                                                                                                                       | https://help.marker.io/en/articles/3738778-webhook-notifications                                                                      |
| Zendesk API token must have permissions for user management, ticket creation, and ticket updating to ensure workflow success.                                                                                           | Credential setup requirements                                                                                                        |
| Troubleshooting common issues such as webhook not triggering, user creation failure, ticket creation failure, internal comment missing, and custom field updates not working.                                             | See Sticky Note3 content                                                                                                            |
| Replace `[REPLACE_SUBDOMAIN]` in all Zendesk API URLs with your actual Zendesk subdomain before activating the workflow.                                                                                                  | Important configuration step                                                                                                        |
| Custom field ID in the "Create Ticket" node must match your Zendesk ticket field ID configured to store Marker ID or other custom data.                                                                                   | Customization note                                                                                                                  |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.