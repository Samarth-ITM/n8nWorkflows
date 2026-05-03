User-Friendly Technical Support Portal for GLPI Ticket Management

https://n8nworkflows.xyz/workflows/user-friendly-technical-support-portal-for-glpi-ticket-management-7392


# User-Friendly Technical Support Portal for GLPI Ticket Management

### 1. Workflow Overview

This workflow implements a **User-Friendly Technical Support Portal** integrated with the GLPI ticket management system. It is designed to streamline the submission and processing of IT support requests and incidents by end-users, providing a custom interface alternative to the standard GLPI UI. The workflow handles two main types of user inputs — *Requests* and *Incidents* — and manages the ticket creation, file attachment uploads, and session handling with GLPI via its REST API.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception:** Captures the type of issue and user contact information through forms.
- **1.2 Session Initialization:** Authenticates with the GLPI API to obtain a session token.
- **1.3 Request or Incident Branching:** Routes the flow based on user selection (Request or Incident).
- **1.4 Request Handling:** Collects detailed request info, maps categories/priorities, uploads authorization files if provided, creates the ticket in GLPI, and links uploaded files.
- **1.5 Incident Handling:** Collects detailed incident info, maps categories/urgency, uploads evidence files if provided, creates the incident ticket in GLPI, and links uploaded files.
- **1.6 Ticket Finalization and Sign-Out:** Completes the user interaction with success messages and terminates the GLPI session.
- **1.7 No Operation Node:** Final sink node to end the workflow cleanly.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Reception

**Overview:**  
This block gathers initial user input on the type of support needed and collects their contact information for follow-up.

**Nodes Involved:**  
- Technical Support Portal  
- Configuration Variables  
- Contact information

**Node Details:**

- **Technical Support Portal**  
  - Type: Form Trigger  
  - Role: Entry point; presents a dropdown for users to select "Request" or "Incident".  
  - Configuration: Form with a single required dropdown field labeled "Select the type of request to make" with options "Request" and "Incident". Includes descriptive text explaining the difference between requests and incidents.  
  - Input: External HTTP webhook trigger.  
  - Output: User selection passed downstream.

- **Configuration Variables**  
  - Type: Set  
  - Role: Stores GLPI server URL and app token used throughout the workflow.  
  - Configuration: Two string fields: `glpi_url` and `app_token`.  
  - Inputs: Output from previous node.  
  - Outputs: Variables injected for HTTP requests.

- **Contact information**  
  - Type: Form  
  - Role: Collects user’s name, phone number, and email for ticket communication.  
  - Configuration: Form fields with required Name (text), Phone (number), and Email (email format).  
  - Input: Output from Configuration Variables node.  
  - Output: Contact details JSON.

**Potential Failures:**  
- User might submit invalid contact info formats or omit required fields (handled by form validation).  
- Configuration Variables must be correctly set; otherwise, authentication fails downstream.

---

#### 1.2 Session Initialization

**Overview:**  
Authenticates with the GLPI API using basic HTTP authentication and application token, retrieves a session token to authorize subsequent API calls.

**Nodes Involved:**  
- Get session token

**Node Details:**

- **Get session token**  
  - Type: HTTP Request  
  - Role: Initiates GLPI session by POSTing to `/initSession` endpoint.  
  - Configuration: HTTP Basic Auth credentials (configured externally in n8n), headers include `App-Token` and `Content-Type: application/json`.  
  - Input: Contact information node output.  
  - Output: JSON containing `session_token`.  
  - Version: HTTP Request node version 4.1+ (supports auth and headers).  
  - Failure Modes: Authentication errors (wrong credentials or app token), network timeouts.

---

#### 1.3 Request or Incident Branching

**Overview:**  
Branches workflow logic depending on whether the user selected "Request" or "Incident".

**Nodes Involved:**  
- 1. Request or incident

**Node Details:**

- **1. Request or incident**  
  - Type: Switch  
  - Role: Routes flow to Request or Incident sub-flows based on user selection.  
  - Configuration: Two conditions checking if the value from "Technical Support Portal" dropdown equals "Request" or "Incident".  
  - Input: Session token node output.  
  - Output: Two outputs labeled "Request" and "Incident".  
  - Failure Modes: Expression failure if input JSON path is incorrect or missing.

---

#### 1.4 Request Handling

**Overview:**  
Gathers detailed request info, maps category and priority to internal IDs, optionally uploads authorization files, creates the GLPI ticket, links uploaded files, and finalizes with confirmation.

**Nodes Involved:**  
- Request Form  
- 1. Check if File Exists  
- 1. Upload File  
- Request Category ID  
- Create Request  
- If1  
- 1. Link File to Ticket  
- Finish Request

**Node Details:**

- **Request Form**  
  - Type: Form  
  - Role: Collects request details: Title, Description, Category (multi-select dropdown), Priority (dropdown), and optional authorization file attachment.  
  - Configuration: Required fields for title, description, category (multi-select among Licenses, Computer, Peripherals), and priority (Low, Medium, High).  
  - Input: Request branch output from switch.  
  - Output: JSON with user inputs and possibly a binary file.

- **1. Check if File Exists**  
  - Type: If  
  - Role: Checks if the user attached any file in the authorization field.  
  - Configuration: Condition checks if the binary property in the Request Form node is not empty.  
  - Input: Request Form output.  
  - Output: Two outputs - true (file exists) leads to upload, false skips upload.

- **1. Upload File**  
  - Type: HTTP Request (multipart form-data)  
  - Role: Uploads the authorization file to GLPI's Document endpoint.  
  - Configuration: Sends multipart form with `uploadManifest` JSON describing the file and the file binary data. Headers include session and app tokens.  
  - Input: True branch of 1. Check if File Exists.  
  - Output: JSON response with uploaded document ID.

- **Request Category ID**  
  - Type: Code  
  - Role: Maps the selected category and priority text to GLPI internal category and priority IDs used in ticket creation.  
  - Configuration: JavaScript switch-case mapping categories (Computer=6, Licenses=7, Peripherals=8) and priorities (Low=2, Medium=3, High=4).  
  - Input: From 1. Upload File (if file uploaded) or directly from 1. Check if File Exists false branch.  
  - Output: Object with `id` (category ID) and `priority` (priority ID).

- **Create Request**  
  - Type: HTTP Request  
  - Role: Creates a new ticket of type Request (type=2) in GLPI using collected info, including concatenated contact info in ticket content.  
  - Configuration: POST to `/Ticket/` with JSON body containing title, content, type, status, priority, category ID, and placeholders for recipient and entity IDs. Headers include session and app tokens.  
  - Input: Request Category ID output.  
  - Output: GLPI ticket creation response.

- **If1**  
  - Type: If  
  - Role: Checks if file upload was executed to decide whether to link the uploaded file to the ticket.  
  - Configuration: Checks if the 1. Upload File node executed.  
  - Input: Create Request output.  
  - Output: True leads to linking file, false leads directly to finalization.

- **1. Link File to Ticket**  
  - Type: HTTP Request  
  - Role: Links the uploaded document to the created ticket in GLPI by creating a Document_Item record.  
  - Configuration: POST to `/Document_Item/` with JSON body containing document ID, itemtype "Ticket", and ticket ID from Create Request. Headers include session and app tokens.  
  - Input: True branch of If1.  
  - Output: Response confirming link.

- **Finish Request**  
  - Type: Form (Completion)  
  - Role: Displays a completion message with the created ticket ID to the user.  
  - Configuration: Completion title "Your request was created successfully", message includes ticket ID from Create Request.  
  - Input: Either from 1. Link File to Ticket or If1 false branch.  
  - Output: Leads to sign out.

---

#### 1.5 Incident Handling

**Overview:**  
Handles incident tickets similarly to requests but with incident-specific category and urgency mappings, evidence file upload, ticket creation, file linking, and confirmation.

**Nodes Involved:**  
- Incident Form  
- 2. Check if File Exists  
- 2. Upload File  
- Incident Category ID  
- Create Incident  
- If  
- 2. Link File to Ticket  
- Finish Incident

**Node Details:**

- **Incident Form**  
  - Type: Form  
  - Role: Collects incident details: Title, Description, Category (multi-select), Urgency (dropdown), and optional evidence file attachment.  
  - Configuration: Required fields similar to request form but categories are incident-specific: "Computer does not turn on", "Server down", "Massive website failure". Urgency levels: Low, Medium, High.  
  - Input: Incident branch output from switch.  
  - Output: JSON with user inputs and possibly a binary file.

- **2. Check if File Exists**  
  - Type: If  
  - Role: Checks for evidence file attachment.  
  - Configuration: Checks if binary data exists in Incident Form output.  
  - Input: Incident Form output.  
  - Output: True leads to upload, false bypasses.

- **2. Upload File**  
  - Type: HTTP Request (multipart form-data)  
  - Role: Uploads evidence file to GLPI Document endpoint.  
  - Configuration: Similar to 1. Upload File but uses evidence field.  
  - Input: True branch of 2. Check if File Exists.  
  - Output: Uploaded document ID.

- **Incident Category ID**  
  - Type: Code  
  - Role: Maps incident category and urgency text to GLPI IDs for category and urgency.  
  - Configuration: JavaScript with switch-case for categories (mapping to IDs 9, 10, 11) and urgency (Low=2, Medium=3, High=4).  
  - Input: 2. Upload File output or 2. Check if File Exists false branch.  
  - Output: Object with category ID and urgency.

- **Create Incident**  
  - Type: HTTP Request  
  - Role: Creates an incident ticket (type=1) in GLPI with user info and mapped fields.  
  - Configuration: POST to `/Ticket/` with JSON body including title, content, type, status, urgency, category ID.  
  - Input: Incident Category ID output.  
  - Output: GLPI ticket response.

- **If**  
  - Type: If  
  - Role: Checks if file upload executed to decide linking.  
  - Configuration: Checks execution of 2. Upload File node.  
  - Input: Create Incident output.  
  - Output: True leads to link, false skips.

- **2. Link File to Ticket**  
  - Type: HTTP Request  
  - Role: Links uploaded evidence document to created ticket.  
  - Configuration: POST to `/Document_Item/` with document ID and ticket ID.  
  - Input: True branch of If.  
  - Output: Link confirmation.

- **Finish Incident**  
  - Type: Form (Completion)  
  - Role: Shows success message with ticket ID.  
  - Configuration: Completion title and message with ticket ID.  
  - Input: Either from 2. Link File to Ticket or If false branch.  
  - Output: Leads to sign out.

---

#### 1.6 Ticket Finalization and Sign-Out

**Overview:**  
Finalizes the user interaction by signing out from the GLPI session and ending the workflow cleanly.

**Nodes Involved:**  
- Sign Out GLPI  
- No Operation, do nothing

**Node Details:**

- **Sign Out GLPI**  
  - Type: HTTP Request  
  - Role: Sends a request to GLPI `/killSession` endpoint to invalidate the session token.  
  - Configuration: Headers include current session and app tokens.  
  - Input: From Finish Request or Finish Incident node.  
  - Output: Response from GLPI.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Terminal node for workflow end, performs no action.  
  - Input: From Sign Out GLPI.  
  - Output: None.

---

#### 1.7 Sticky Notes (Documentation)

**Overview:**  
Provides user-facing documentation and setup instructions directly in the workflow canvas.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - Content: Describes the purpose and advantages of the custom GLPI interface created with n8n.  
  - No inputs/outputs.

- **Sticky Note1**  
  - Content: Lists requirements (GLPI REST API, admin privileges), setup instructions for URL and tokens, and category ID mapping explanations.  
  - No inputs/outputs.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                       | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                       |
|--------------------------|--------------------|------------------------------------|------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Technical Support Portal  | Form Trigger       | Entry form for selecting request type | -                            | Configuration Variables         |                                                                                                 |
| Configuration Variables   | Set                | Stores GLPI URL and app token       | Technical Support Portal      | Contact information             |                                                                                                 |
| Contact information       | Form               | Collects user contact details       | Configuration Variables       | Get session token              |                                                                                                 |
| Get session token         | HTTP Request       | Authenticates and obtains session token | Contact information          | 1. Request or incident          |                                                                                                 |
| 1. Request or incident    | Switch             | Routes to Request or Incident flows | Get session token             | Request Form, Incident Form     |                                                                                                 |
| Request Form             | Form               | Collects detailed request info      | 1. Request or incident        | 1. Check if File Exists         |                                                                                                 |
| 1. Check if File Exists   | If                 | Checks for authorization file upload | Request Form                | 1. Upload File, Request Category ID |                                                                                                 |
| 1. Upload File            | HTTP Request       | Uploads authorization file          | 1. Check if File Exists       | Request Category ID             |                                                                                                 |
| Request Category ID       | Code               | Maps category and priority to GLPI IDs | 1. Upload File / 1. Check if File Exists | Create Request           |                                                                                                 |
| Create Request            | HTTP Request       | Creates request ticket in GLPI      | Request Category ID           | If1                           |                                                                                                 |
| If1                      | If                 | Checks if authorization file uploaded to link | Create Request            | 1. Link File to Ticket, Finish Request |                                                                                                 |
| 1. Link File to Ticket    | HTTP Request       | Links authorization file to ticket  | If1                          | Finish Request                 |                                                                                                 |
| Finish Request            | Form (Completion)  | Shows success message for request   | 1. Link File to Ticket, If1   | Sign Out GLPI                 |                                                                                                 |
| Incident Form             | Form               | Collects detailed incident info     | 1. Request or incident        | 2. Check if File Exists         |                                                                                                 |
| 2. Check if File Exists   | If                 | Checks for evidence file upload     | Incident Form                | 2. Upload File, Incident Category ID |                                                                                                 |
| 2. Upload File            | HTTP Request       | Uploads evidence file                | 2. Check if File Exists       | Incident Category ID            |                                                                                                 |
| Incident Category ID      | Code               | Maps category and urgency to GLPI IDs | 2. Upload File / 2. Check if File Exists | Create Incident          |                                                                                                 |
| Create Incident           | HTTP Request       | Creates incident ticket in GLPI     | Incident Category ID          | If                            |                                                                                                 |
| If                       | If                 | Checks if evidence file uploaded to link | Create Incident            | 2. Link File to Ticket, Finish Incident |                                                                                                 |
| 2. Link File to Ticket    | HTTP Request       | Links evidence file to ticket        | If                           | Finish Incident                |                                                                                                 |
| Finish Incident           | Form (Completion)  | Shows success message for incident  | 2. Link File to Ticket, If    | Sign Out GLPI                 |                                                                                                 |
| Sign Out GLPI             | HTTP Request       | Ends GLPI session                   | Finish Request, Finish Incident | No Operation, do nothing        |                                                                                                 |
| No Operation, do nothing  | NoOp               | Ends workflow cleanly               | Sign Out GLPI                | -                              |                                                                                                 |
| Sticky Note               | Sticky Note        | Workflow purpose and interface info | -                            | -                              | ## Custom interface for GLPI with n8n: Create custom templates for your clients...               |
| Sticky Note1              | Sticky Note        | Setup requirements and instructions | -                            | -                              | Requirements: GLPI REST API, admin privileges; Update GLPI server URL and app token...           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Technical Support Portal" node**  
   - Type: Form Trigger  
   - Configure webhook with a unique ID.  
   - Add a dropdown field labeled "Select the type of request to make:" with options: Request, Incident (required).  
   - Add descriptive text explaining "Request" vs "Incident".  
   - Position near the start of the canvas.

2. **Create "Configuration Variables" node**  
   - Type: Set  
   - Add two string fields:  
     - `glpi_url`: your GLPI server base URL (e.g., `https://your_glpi_server.com`)  
     - `app_token`: your GLPI application token  
   - Connect output of "Technical Support Portal" to this node.

3. **Create "Contact information" form node**  
   - Fields:  
     - Name (text, required, placeholder "Juan Gomez")  
     - Phone (number, required, placeholder "3123456789")  
     - Email (email, required, placeholder "jgomez@email.com")  
   - Connect from Configuration Variables.

4. **Create "Get session token" HTTP Request node**  
   - Method: POST  
   - URL: `={{ $json.glpi_url }}/apirest.php/initSession`  
   - Authentication: HTTP Basic Auth with GLPI user credentials (configure in n8n credentials).  
   - Headers:  
     - `App-Token`: from Configuration Variables  
     - `Content-Type`: `application/json`  
   - Connect from Contact information.

5. **Create "1. Request or incident" switch node**  
   - Conditions:  
     - Output "Request" if `={{ $('Technical Support Portal').item.json['Select the type of request to make:'] }}` equals "Request"  
     - Output "Incident" if equals "Incident"  
   - Connect from Get session token.

6. **Request branch:**  
   - Create "Request Form" node: form with fields Title, Description (textarea), Category (multi-select dropdown: Licenses, Computer, Peripherals), Priority (dropdown: Low, Medium, High), Attach authorization (file, optional).  
   - Connect from switch "Request" output.

   - Create "1. Check if File Exists" If node:  
     - Condition: check if `$('Request Form').item.binary` is not empty.  
   - Connect from Request Form.

   - Create "1. Upload File" HTTP Request node:  
     - Method: POST  
     - URL: `={{ $json.glpi_url }}/apirest.php/Document/`  
     - Content-Type: multipart/form-data  
     - Headers: `Session-Token` and `App-Token` from previous nodes.  
     - Body parameters: multipart with `uploadManifest` JSON and binary data from Attach authorization file.  
   - Connect true branch from 1. Check if File Exists.

   - Create "Request Category ID" Code node:  
     - JavaScript mapping category text and priority text to IDs (Computer=6, Licenses=7, Peripherals=8; Low=2, Medium=3, High=4).  
   - Connect from 1. Upload File true branch and also from 1. Check if File Exists false branch (to handle no file upload).

   - Create "Create Request" HTTP Request node:  
     - POST to `/Ticket/` with JSON body: fields include name, content (concatenate description and contact info), type=2, status=1, priority and category IDs.  
     - Headers: session and app tokens.  
   - Connect from Request Category ID.

   - Create "If1" node:  
     - Condition: check if "1. Upload File" executed.  
   - Connect from Create Request.

   - Create "1. Link File to Ticket" HTTP Request node:  
     - POST to `/Document_Item/` with JSON body including document ID from "1. Upload File" and ticket ID from Create Request.  
     - Headers: session and app tokens.  
   - Connect true branch from If1.

   - Create "Finish Request" form node (completion):  
     - Completion title and message showing ticket ID from Create Request.  
   - Connect from "1. Link File to Ticket" and false branch of If1.

7. **Incident branch:**  
   - Create "Incident Form" node: form similar to Request Form but fields are Title, Description, Category (multi-select: Computer does not turn on, Server down, Massive website failure), Urgency (Low, Medium, High), Attach evidence (file optional).  
   - Connect from switch "Incident" output.

   - Create "2. Check if File Exists" If node:  
     - Condition: check if `$('Incident Form').item.binary` is not empty.  
   - Connect from Incident Form.

   - Create "2. Upload File" HTTP Request node:  
     - Similar to 1. Upload File but uses evidence file field.  
   - Connect true branch from 2. Check if File Exists.

   - Create "Incident Category ID" Code node:  
     - JavaScript mapping categories (Computer does not turn on=9, Server down=10, Massive website failure=11) and urgency (Low=2, Medium=3, High=4).  
   - Connect from 2. Upload File true branch and 2. Check if File Exists false branch.

   - Create "Create Incident" HTTP Request node:  
     - POST to `/Ticket/` with JSON body including incident details, type=1, status=1, urgency and category IDs, and concatenated contact info.  
   - Connect from Incident Category ID.

   - Create "If" node:  
     - Condition: check if "2. Upload File" executed.  
   - Connect from Create Incident.

   - Create "2. Link File to Ticket" HTTP Request node:  
     - POST to `/Document_Item/` to link uploaded evidence document to the ticket.  
   - Connect true branch from If node.

   - Create "Finish Incident" form node (completion):  
     - Completion message with ticket ID.  
   - Connect from "2. Link File to Ticket" and false branch of If.

8. **Sign Out and End:**  
   - Create "Sign Out GLPI" HTTP Request node:  
     - POST to `/killSession` with session and app tokens headers to terminate session.  
   - Connect from both Finish Request and Finish Incident nodes.

   - Create "No Operation, do nothing" node:  
     - Type: NoOp  
   - Connect from Sign Out GLPI node.

9. **Sticky Notes:** (Optional for documentation)  
   - Add sticky notes with instructions, requirements, and project description on the canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Custom interface for GLPI with n8n: Create custom templates for your clients and request the specific information needed to address their requests or incidents. Enjoy a revamped interface, more intuitive and functional than the standard one, designed to optimize ticket management. | Workflow purpose description (Sticky Note)                                                            |
| Requirements: GLPI REST API, user with app admin privileges. Update GLPI server URL and app token in "Configuration Variables" node. Categories must be mapped to their correct IDs visible in GLPI UI (Setup > Dropdowns > Assistance > ITIL categories).                                                        | Setup instructions (Sticky Note1)                                                                     |

---

**Disclaimer:**  
The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.