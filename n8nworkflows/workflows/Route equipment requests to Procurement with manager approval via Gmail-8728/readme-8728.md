Route equipment requests to Procurement with manager approval via Gmail

https://n8nworkflows.xyz/workflows/route-equipment-requests-to-procurement-with-manager-approval-via-gmail-8728


# Route equipment requests to Procurement with manager approval via Gmail

---

### 1. Workflow Overview

This workflow automates the routing of equipment requests submitted by employees, enforcing manager approval when required, and notifying the Procurement department accordingly via Gmail. It is designed for organizations where employees may or may not have a manager assigned, and approval paths differ accordingly.

The workflow consists of the following logical blocks:

- **1.1 Employee Identification and Validation**: Receives an employee enrollment number, validates it against an employee database, and handles cases where the employee is not found.

- **1.2 Equipment Request Collection**: Gathers detailed equipment request information from the validated employee.

- **1.3 Approval Decision and Manager Retrieval**: Determines if the employee requires manager approval and fetches the manager's details if needed.

- **1.4 Approval Request Composition and Sending**: Uses AI to sanitize and professionally format the request, then sends an approval request email to the manager, awaiting their decision.

- **1.5 Manager Decision Handling and Notification**: Based on the manager's approval or denial, notifies the employee and, if approved, composes and sends a procurement request email to the purchasing department.

- **1.6 Direct Procurement for Managers**: Allows employees without a manager (assumed managers themselves) to directly request procurement without approval.

- **1.7 Completion and Feedback to Requester**: Presents appropriate feedback forms to the requester based on each path's final outcome.

---

### 2. Block-by-Block Analysis

#### 1.1 Employee Identification and Validation

- **Overview**: Receives the employee's enrollment number via a form, looks up the employee record in the database, and branches based on whether the employee exists.

- **Nodes Involved**:  
  - Question form  
  - Get Employee  
  - If (employee found?)  
  - Return Employee not found  

- **Node Details**:

  - **Question form**  
    - Type: Form Trigger  
    - Role: Entry point for employee enrollment input  
    - Configuration: Single required field labeled "Your employee enrollment" with custom CSS for styling and a submit button labeled "Send question"  
    - Inputs: None (triggered by form submission)  
    - Outputs: Employee enrollment number in JSON  
    - Edge Cases: Invalid or blank enrollment; user cancels form  
    - Notes: Custom styling ensures consistent UX  

  - **Get Employee**  
    - Type: MySQL (Select)  
    - Role: Queries employee database table for record matching the provided enrollment number  
    - Configuration: Selects from `employees` table where `enrollment_number` equals the submitted value; returns all matches  
    - Inputs: Enrollment number from "Question form"  
    - Outputs: Employee record(s) or empty result  
    - Edge Cases: Database connection issues, no matching record, multiple matches (unlikely if enrollment unique)  
    - Version: v2.5 recommended for MySQL node  
    - Credentials: MySQL credentials named "MySQL - Estudos"  
    - Notes: Must handle empty results downstream  

  - **If**  
    - Type: Conditional (If)  
    - Role: Checks if the employee query returned empty (no employee found)  
    - Configuration: Condition testing `$json.isEmpty()` equals true  
    - Inputs: Output from "Get Employee"  
    - Outputs: Two branches: TRUE (empty, no employee found), FALSE (employee found)  
    - Edge Cases: Expression evaluation failure if input malformed  

  - **Return Employee not found**  
    - Type: Form (Completion)  
    - Role: User-facing form completion page shown when no employee matches enrollment  
    - Configuration: Title "Employee not found" and message informing the user about the failure to find enrollment  
    - Inputs: TRUE branch of "If"  
    - Outputs: None (workflow ends here)  
    - Edge Cases: None specific; message customization recommended  

#### 1.2 Equipment Request Collection

- **Overview**: After successful employee validation, collects the details of the equipment request via a second form.

- **Nodes Involved**:  
  - Employee data  
  - Get request information  
  - Merge Employee data and request  

- **Node Details**:

  - **Employee data**  
    - Type: Set  
    - Role: Normalizes and structures employee data for consistent downstream use  
    - Configuration: Sets fields `employee.enrollment_number`, `employee.name`, `employee.email`, `employee.manager`, and a boolean `employee.requires_approval` (true if manager exists)  
    - Inputs: Employee record from "Get Employee"  
    - Outputs: Structured employee object  
    - Edge Cases: Missing fields in DB record could cause missing data downstream  

  - **Get request information**  
    - Type: Form Trigger  
    - Role: Collects free-text description of the equipment request from the employee  
    - Configuration: Single required textarea field labeled "What would you like to request?" with consistent custom CSS styling  
    - Inputs: Output from "Employee data" (triggered next)  
    - Outputs: Request text in JSON  
    - Edge Cases: Empty or malicious input; sanitization done later  

  - **Merge Employee data and request**  
    - Type: Merge (Combine)  
    - Role: Combines employee data and equipment request text into a single JSON object for easier processing  
    - Configuration: Combine inputs by position (paired streams)  
    - Inputs: Outputs from "Employee data" and "Get request information"  
    - Outputs: Combined JSON with employee and request details  
    - Edge Cases: If one input missing, merge fails or incomplete data downstream  

#### 1.3 Approval Decision and Manager Retrieval

- **Overview**: Determines if approval is required (employee has a manager). If yes, fetches the manager's data from the database; otherwise, proceeds to procurement directly.

- **Nodes Involved**:  
  - Is approval required?  
  - Get Manager  
  - Set Manager field  
  - Employee and Manager  

- **Node Details**:

  - **Is approval required?**  
    - Type: If  
    - Role: Checks if the employee has a non-empty `employee.manager` value to decide approval path  
    - Configuration: Condition checks if `employee.manager` is not empty  
    - Inputs: Output from "Employee data"  
    - Outputs: TRUE branch (needs approval), FALSE branch (no approval)  
    - Edge Cases: Null or malformed manager field  

  - **Get Manager**  
    - Type: MySQL (Select)  
    - Role: Retrieves the manager’s employee record by ID  
    - Configuration: Select from `employees` where `id` equals `employee.manager`  
    - Inputs: TRUE branch of "Is approval required?"  
    - Outputs: Manager record(s)  
    - Credentials: Same MySQL as "Get Employee"  
    - Edge Cases: Manager record missing or DB errors  

  - **Set Manager field**  
    - Type: Set  
    - Role: Wraps the manager record into a single `manager` object for convenient access  
    - Configuration: Assigns the entire JSON of the manager query result to a `manager` field  
    - Inputs: Output from "Get Manager"  
    - Outputs: JSON with `manager` object  
    - Edge Cases: Empty or multiple records could affect structure  

  - **Employee and Manager**  
    - Type: Merge (Combine)  
    - Role: Combines employee data and wrapped manager object for consolidated downstream use  
    - Configuration: Combine inputs by position  
    - Inputs: From "Set Manager field" and from "Is approval required?" FALSE branch (which has employee data only)  
    - Outputs: JSON containing both employee and manager info (or only employee if no manager)  
    - Edge Cases: Missing inputs cause incomplete data  

#### 1.4 Approval Request Composition and Sending

- **Overview**: For employees requiring approval, generates a polished approval email using AI, sends it to the manager via Gmail, and waits for approval response.

- **Nodes Involved**:  
  - Approval request message  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Data and message  
  - Send request to approve  

- **Node Details**:

  - **Approval request message**  
    - Type: If  
    - Role: Gate node to decide if approval email generation is needed (checks `employee.requires_approval` boolean)  
    - Inputs: Output from "Merge Employee data and request"  
    - Outputs: TRUE triggers approval message generation; FALSE triggers direct procurement message  
    - Edge Cases: Incorrect boolean could misroute flow  

  - **OpenAI Chat Model**  
    - Type: AI language model (OpenAI GPT-4.1 nano)  
    - Role: Generates a sanitized, professional approval email text and a sanitized version of the request  
    - Configuration: Uses prompt to instruct AI to fix grammar, remove informal language, and produce JSON output with `sanitized_request` and `email_text` addressed to the manager  
    - Inputs: Employee request text and context from "Approval request message"  
    - Credentials: OpenAI API credential named "OpenAi account"  
    - Edge Cases: API key limits, network timeouts, unexpected AI output format  

  - **Structured Output Parser**  
    - Type: AI output parser  
    - Role: Parses the AI response JSON strictly to extract the email text and sanitized request  
    - Inputs: Raw AI model output  
    - Outputs: Parsed structured JSON  
    - Edge Cases: Parsing failures if AI response malformed  

  - **Data and message**  
    - Type: Merge (Combine)  
    - Role: Combines employee/manager/request context with sanitized AI output for final email sending  
    - Inputs: Output of AI model parsing and employee-manager-request data  
    - Outputs: Fully composed message JSON for sending  
    - Edge Cases: Missing input causes incomplete data  

  - **Send request to approve**  
    - Type: Gmail (Send and Wait for Approval)  
    - Role: Sends the approval email to the manager and waits for double opt-in approval or denial  
    - Configuration: Sends email to `manager.email` with subject referencing employee name and enrollment, message body from AI-generated `email_text`, waits for manager decision  
    - Credentials: Gmail OAuth2 credential with send permission  
    - Inputs: Approval email content from "Data and message"  
    - Outputs: Approval result (`data.approved` true/false)  
    - Edge Cases: Gmail API errors, manager not responding, email deliverability issues  

#### 1.5 Manager Decision Handling and Notification

- **Overview**: Routes the workflow based on manager's approval decision. If approved, notifies employee and sends a procurement email; if denied, notifies employee accordingly.

- **Nodes Involved**:  
  - Request approved?  
  - Notify approved request  
  - Create request approved message  
  - Structured Output Parser2  
  - Send request approved to the purchasing department  
  - Notify denied request  
  - End employee request  
  - End manager request  

- **Node Details**:

  - **Request approved?**  
    - Type: If  
    - Role: Checks if `data.approved` from Gmail approval email node is true (approved) or false (denied)  
    - Inputs: Output from "Send request to approve"  
    - Outputs: TRUE branch for approved, FALSE for denied  
    - Edge Cases: Missing or malformed approval data  

  - **Notify approved request**  
    - Type: Gmail (Send)  
    - Role: Sends confirmation email to the employee that their request was approved and forwarded to Procurement  
    - Configuration: Sends to `employee.email` with a polite approval confirmation message including sanitized request details  
    - Credentials: Gmail OAuth2  
    - Inputs: From merged data including employee info and sanitized request  
    - Edge Cases: Email failures, invalid employee email  

  - **Create request approved message**  
    - Type: AI Chain LLM (LangChain)  
    - Role: Generates a polished procurement email addressed to the Purchasing Department, explicitly stating manager approval  
    - Configuration: Similar prompt style to approval message, but for procurement email, returns JSON with sanitized request and email text  
    - Inputs: Employee request and context from "Data and message"  
    - Credentials: OpenAI API  
    - Edge Cases: API or parsing failures  

  - **Structured Output Parser2**  
    - Type: AI output parser  
    - Role: Parses AI response JSON for procurement email output  
    - Inputs: Raw AI output from "Create request approved message"  
    - Outputs: Parsed JSON with email text and sanitized request  
    - Edge Cases: Parsing errors  

  - **Send request approved to the purchasing department**  
    - Type: No Operation placeholder (to be replaced)  
    - Role: Represents sending the procurement email to the purchasing team (could be replaced with Gmail, ticket system, Slack, etc.)  
    - Inputs: Parsed procurement email text from AI output parser  
    - Outputs: None  
    - Edge Cases: Replace with actual sending node  

  - **Notify denied request**  
    - Type: Gmail (Send)  
    - Role: Sends notification to the employee that the manager denied the request  
    - Configuration: Similar to approval notification but with denial message and instructions if needed  
    - Inputs: From "Data and message"  
    - Credentials: Gmail OAuth2  
    - Edge Cases: Email deliverability, invalid employee email  

  - **End employee request**  
    - Type: Form (Completion)  
    - Role: Shows confirmation page to employee after submitting request for manager approval  
    - Inputs: After sending approval email  
    - Outputs: None  
    - Edge Cases: None  

  - **End manager request**  
    - Type: Form (Completion)  
    - Role: Shows confirmation page to employee when request is directly sent to Procurement (no manager approval)  
    - Inputs: After sending procurement email for managerless employees  
    - Outputs: None  
    - Edge Cases: None  

#### 1.6 Direct Procurement for Managers (No Approval Required)

- **Overview**: Employees without a manager (likely managers themselves) skip approval and directly submit a procurement request email.

- **Nodes Involved**:  
  - Create request message  
  - OpenAI Chat Model1  
  - Structured Output Parser1  
  - Send request to the purchasing department  
  - End manager request  

- **Node Details**:

  - **Create request message**  
    - Type: AI Chain LLM (LangChain)  
    - Role: Sanitizes and formats the equipment request into a professional procurement email without manager approval mention  
    - Inputs: Employee request and context  
    - Configuration: Prompt asks AI to produce JSON with sanitized request and procurement email text addressed to Purchasing Department  
    - Credentials: OpenAI API  
    - Edge Cases: API errors, malformed response  

  - **OpenAI Chat Model1**  
    - Type: OpenAI GPT-4.1 nano  
    - Role: Runs the AI prompt for direct procurement message creation  
    - Inputs: From "Create request message"  
    - Outputs: Raw AI output  
    - Credentials: OpenAI API  
    - Edge Cases: Timeout or key errors  

  - **Structured Output Parser1**  
    - Type: AI output parser  
    - Role: Parses AI output into structured JSON  
    - Inputs: Raw AI output  
    - Outputs: Parsed email content  
    - Edge Cases: Parsing failure  

  - **Send request to the purchasing department**  
    - Type: No Operation placeholder  
    - Role: Represents sending procurement email to purchasing team  
    - Inputs: Parsed email content  
    - Edge Cases: To be replaced with actual sending node  

  - **End manager request**  
    - Type: Form (Completion)  
    - Role: Confirmation page for employee after direct procurement request submission  
    - Inputs: After sending procurement request  
    - Edge Cases: None  

#### 1.7 Completion and Feedback to Requester

- **Overview**: Displays final feedback forms to the user depending on the workflow path: employee not found, request sent for approval, request sent directly to procurement, or after approval/denial.

- **Nodes Involved**:  
  - Return Employee not found  
  - End employee request  
  - End manager request  

- **Node Details**:

  - **Return Employee not found**  
    - Shows error message when employee enrollment not found.  

  - **End employee request**  
    - Displays confirmation that request has been sent for approval and sets expectations.  

  - **End manager request**  
    - Displays confirmation that request was sent directly to procurement.  

---

### 3. Summary Table

| Node Name                        | Node Type                           | Functional Role                                    | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                                         |
|---------------------------------|-----------------------------------|--------------------------------------------------|-------------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Question form                   | Form Trigger                      | Collect employee enrollment number                | None                                | Get Employee                        | Collects the employee enrollment number. Tip: make label clear; consider help text and validation regex.                           |
| Get Employee                   | MySQL                             | Lookup employee record by enrollment              | Question form                      | If                                 | Looks up employee by enrollment_number in `employees` table. Expected columns: id, name, email, enrollment_number, manager (nullable).|
| If                             | If                                | Check if employee found                            | Get Employee                      | Return Employee not found, Employee data | Checks if DB lookup returned empty. TRUE → show "Employee not found", FALSE → continue.                                           |
| Return Employee not found      | Form Completion                   | Show error when no employee found                  | If (TRUE)                        | None                                | User-facing message when enrollment number not matched. Customize with support info.                                               |
| Employee data                 | Set                                | Normalize employee data fields                     | If (FALSE)                      | Get request information, Is approval required? | Normalizes employee object for downstream use, includes requires_approval boolean.                                                |
| Get request information       | Form Trigger                      | Collect equipment request details                   | Employee data                    | Merge Employee data and request      | Asks employee what they want to request; uses consistent CSS styling.                                                              |
| Merge Employee data and request| Merge (Combine)                   | Combine employee info and request text             | Employee data, Get request information | Approval request message            | Combines employee context and free-text request data.                                                                              |
| Is approval required?          | If                                | Check if manager approval is required              | Employee data                    | Get Manager (TRUE), Merge Employee data and request (FALSE) | Branches based on manager presence: TRUE fetches manager, FALSE skips to procurement.                                              |
| Get Manager                   | MySQL                             | Fetch manager record by ID                          | Is approval required? (TRUE)     | Set Manager field                   | Fetches manager record from `employees` by id. Expected columns: id, name, email.                                                  |
| Set Manager field             | Set                                | Wrap manager record into a single object           | Get Manager                     | Employee and Manager                | Wraps manager row into single `manager` object for ease of access.                                                                 |
| Employee and Manager          | Merge (Combine)                   | Combine employee and manager data                   | Set Manager field, Is approval required? (FALSE) | Merge Employee data and request    | Merges employee and manager streams after DB lookup.                                                                               |
| Approval request message      | If                                | Gate for approval email generation                  | Merge Employee data and request   | Create approval message (TRUE), Create request message (FALSE) | Simple gate: TRUE → build approval email, FALSE → build procurement email.                                                        |
| OpenAI Chat Model             | AI Language Model (OpenAI GPT)   | Generate sanitized approval request email           | Approval request message          | Structured Output Parser            | Prompts LLM to sanitize request and produce formal approval email JSON.                                                           |
| Structured Output Parser      | AI Output Parser                 | Parse AI JSON output                               | OpenAI Chat Model                | Create approval message             | Parses AI response into structured JSON containing email_text and sanitized_request.                                              |
| Data and message              | Merge (Combine)                   | Combine employee/manager context with AI output     | Create approval message, Merge Employee data and request | End employee request                | Combines sanitized output with context for email sending.                                                                          |
| Send request to approve       | Gmail Send and Wait for Approval | Send approval email to manager, wait for decision  | Data and message                 | Request approved?                  | Emails manager with approval request using Gmail; waits for double opt-in approval.                                               |
| Request approved?             | If                                | Check manager approval decision                     | Send request to approve           | Notify approved request (TRUE), Notify denied request (FALSE) | Routes based on manager decision: TRUE → notify approved + send procurement email, FALSE → notify denied.                         |
| Notify approved request       | Gmail Send                       | Notify employee of approval and forwarding          | Request approved? (TRUE)          | Create request approved message     | Sends confirmation email to employee that request approved and forwarded to Procurement.                                           |
| Create request approved message| AI Chain LLM                     | Generate procurement email for approved request     | Notify approved request           | Structured Output Parser2           | Creates polished procurement email explicitly stating manager approval.                                                           |
| Structured Output Parser2     | AI Output Parser                 | Parse AI JSON procurement email                     | Create request approved message   | Send request approved to the purchasing department | Parses AI output to structured procurement email JSON.                                                                             |
| Send request approved to the purchasing department | No Operation (placeholder)       | Placeholder for sending procurement email           | Structured Output Parser2         | End manager request                | Replace with your procurement notification method (Gmail, ticket, Slack, etc.)                                                    |
| Notify denied request         | Gmail Send                       | Notify employee of denial                             | Request approved? (FALSE)         | End employee request               | Sends email informing employee the request was denied by manager.                                                                  |
| Create request message        | AI Chain LLM                    | Generate procurement email for direct requests (no approval) | Approval request message (FALSE)   | OpenAI Chat Model1                 | Sanitizes and creates procurement email for employees without manager approval.                                                   |
| OpenAI Chat Model1            | AI Language Model (OpenAI GPT)  | Generate direct procurement email                     | Create request message            | Structured Output Parser1          | Runs prompt to create procurement email without approval mention.                                                                  |
| Structured Output Parser1     | AI Output Parser                 | Parse direct procurement email JSON                   | OpenAI Chat Model1               | Send request to the purchasing department | Parses AI output for procurement email.                                                                                            |
| Send request to the purchasing department | No Operation (placeholder)       | Placeholder for sending procurement email           | Structured Output Parser1         | End manager request                | Replace with actual procurement notification node.                                                                                 |
| End employee request          | Form Completion                 | Final confirmation for employee after approval request sent | Send request to approve           | Send request to approve             | Confirms to employee that request was sent to manager for approval.                                                                |
| End manager request           | Form Completion                 | Final confirmation for employee after direct procurement request | Send request to the purchasing department | None                              | Confirms to employee that request was sent directly to Procurement.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Question form" Node**  
   - Type: Form Trigger  
   - Configure with a single required text field labeled "Your employee enrollment"  
   - Apply custom CSS for consistent styling (copy from existing CSS if desired)  
   - Set button label to "Send question"  
   - This node triggers the workflow on form submission.

2. **Create "Get Employee" Node**  
   - Type: MySQL (Select)  
   - Connect input from "Question form"  
   - Configure to select from `employees` table where `enrollment_number` equals `={{ $json["Your employee enrollment"] }}`  
   - Return all results  
   - Set MySQL credentials appropriately (e.g., "MySQL - Estudos").

3. **Create "If" Node** to check if employee found  
   - Connect input from "Get Employee"  
   - Add condition: Check if output is empty (`={{ $json.isEmpty() }}` is true)  
   - TRUE branch: No employee found  
   - FALSE branch: Employee found

4. **Create "Return Employee not found" Node**  
   - Type: Form (Completion)  
   - Connect from TRUE branch of "If"  
   - Set completion title "Employee not found"  
   - Set message "It was not possible to find your registration based on your enrollment number."

5. **Create "Employee data" Node**  
   - Type: Set  
   - Connect from FALSE branch of "If"  
   - Assign fields:  
     - `employee.enrollment_number` = `={{ $json.enrollment_number }}`  
     - `employee.name` = `={{ $json.name }}`  
     - `employee.email` = `={{ $json.email }}`  
     - `employee.manager` = `={{ $json.manager }}`  
     - `employee.requires_approval` = `={{ $json.manager ? true : false }}`

6. **Create "Get request information" Node**  
   - Type: Form Trigger  
   - Connect from "Employee data"  
   - Add a required textarea field labeled "What would you like to request?"  
   - Use same custom CSS styling as "Question form"

7. **Create "Merge Employee data and request" Node**  
   - Type: Merge (Combine by position)  
   - Connect inputs from "Employee data" and "Get request information"  
   - Output combined JSON with employee and request info

8. **Create "Is approval required?" Node**  
   - Type: If  
   - Connect from "Employee data"  
   - Condition: Check if `employee.manager` is not empty  
   - TRUE: Approval required  
   - FALSE: No approval

9. **Create "Get Manager" Node**  
   - Type: MySQL (Select)  
   - Connect from TRUE branch of "Is approval required?"  
   - Select from `employees` where `id` = `={{ $json.employee.manager }}`  
   - Use same MySQL credentials

10. **Create "Set Manager field" Node**  
    - Type: Set  
    - Connect from "Get Manager"  
    - Assign `manager = {{ $json }}` (wrap entire manager record in `manager` field)

11. **Create "Employee and Manager" Node**  
    - Type: Merge (Combine by position)  
    - Connect inputs from "Set Manager field" and FALSE branch of "Is approval required?" (which is employee data)  
    - Output combined employee and manager data

12. **Connect "Employee and Manager" Node output to "Merge Employee data and request" Node**  
    - Combine data for downstream processing

13. **Create "Approval request message" Node**  
    - Type: If  
    - Connect from "Merge Employee data and request"  
    - Condition: Check if `employee.requires_approval` is true  
    - TRUE branch: approval message generation  
    - FALSE branch: direct procurement message generation

14. **Create "OpenAI Chat Model" Node**  
    - Type: AI Language Model (OpenAI GPT-4.1 nano)  
    - Connect from TRUE branch of "Approval request message"  
    - Configure prompt to:  
      - Sanitize employee request text  
      - Create formal approval request email to manager  
      - Output strict JSON with `sanitized_request` and `email_text`  
    - Use OpenAI API credentials

15. **Create "Structured Output Parser" Node**  
    - Connect from "OpenAI Chat Model"  
    - Configure with JSON schema example matching expected output

16. **Create "Data and message" Merge Node**  
    - Connect inputs from "Structured Output Parser" and "Merge Employee data and request"  
    - Combine sanitized message with context for sending

17. **Create "Send request to approve" Node**  
    - Type: Gmail Send and Wait for Approval  
    - Connect from "Data and message"  
    - Configure to send to `manager.email` with subject referencing employee name and enrollment  
    - Use `email_text` from AI output as message body  
    - Use Gmail OAuth2 credentials with send permission  
    - Enable double opt-in approval options

18. **Create "Request approved?" Node**  
    - Type: If  
    - Connect from "Send request to approve"  
    - Condition: Check if `data.approved` is true

19. **Create "Notify approved request" Node**  
    - Type: Gmail Send  
    - Connect from TRUE branch of "Request approved?"  
    - Send email to `employee.email` confirming approval and forwarding to Procurement  
    - Use prepared message including sanitized request  
    - Use Gmail OAuth2 credentials

20. **Create "Create request approved message" Node**  
    - Type: AI Chain LLM (LangChain)  
    - Connect from "Notify approved request"  
    - Configure prompt to create procurement email addressed to Purchasing Department, indicating manager approval  
    - Use OpenAI API credentials

21. **Create "Structured Output Parser2" Node**  
    - Connect from "Create request approved message"  
    - Parse AI JSON output

22. **Create "Send request approved to the purchasing department" Node**  
    - Type: No Operation (placeholder)  
    - Connect from "Structured Output Parser2"  
    - Replace with actual sending node (e.g., Gmail to procurement@company.com, ticket system integration)

23. **Create "Notify denied request" Node**  
    - Type: Gmail Send  
    - Connect from FALSE branch of "Request approved?"  
    - Send denial notification email to employee  
    - Use Gmail OAuth2 credentials

24. **Create "End employee request" Node**  
    - Type: Form Completion  
    - Connect from "Send request to approve"  
    - Shows confirmation page that request sent for manager approval

25. **Create "Create request message" Node**  
    - Type: AI Chain LLM (LangChain)  
    - Connect from FALSE branch of "Approval request message" (no approval required)  
    - Configure prompt to sanitize request and create procurement email without approval mention  
    - Use OpenAI API credentials

26. **Create "OpenAI Chat Model1" Node**  
    - Connect from "Create request message"  
    - Run AI prompt for direct procurement email generation

27. **Create "Structured Output Parser1" Node**  
    - Connect from "OpenAI Chat Model1"  
    - Parse AI output JSON

28. **Create "Send request to the purchasing department" Node**  
    - Type: No Operation (placeholder)  
    - Connect from "Structured Output Parser1"  
    - Replace with actual procurement notification sender

29. **Create "End manager request" Node**  
    - Type: Form Completion  
    - Connect from "Send request to the purchasing department"  
    - Shows confirmation page that request sent directly to Procurement

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses consistent form CSS styling across all forms for a unified user experience.            | See CSS in "Question form" and "Get request information" nodes.                                  |
| AI prompts are designed for formal, concise, and business-appropriate tone; they strictly require JSON output. | Prompts in "Create approval message" and "Create request approved message" nodes.                |
| Gmail OAuth2 credentials must have send permission to managers and employees.                             | Ensure Gmail OAuth2 setup includes required scopes for sending and approval handling.            |
| No hardcoded credentials are used; all sensitive info is secured via n8n credentials.                     | Security best practice maintained.                                                              |
| "Send request approved to the purchasing department" and "Send request to the purchasing department" nodes are placeholders and need to be replaced with actual sending nodes depending on your organization's tools (e.g., Gmail, Jira, Slack). | Customize as needed to integrate with your procurement notification channels.                    |
| Employee PII is handled carefully and not exposed in logs or unsecured outputs.                           | Sanitization and minimal data exposure enforced by design.                                      |
| For more details on forms and styling, consult n8n's Form Trigger node documentation.                     | https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger/                                            |
| For Gmail approval emails with double opt-in, see n8n Gmail node documentation.                           | https://docs.n8n.io/nodes/n8n-nodes-base.gmail/                                                 |
| For LangChain and OpenAI nodes, see n8n AI integration docs.                                              | https://docs.n8n.io/integrations/ai/                                                            |

---

**Disclaimer**:  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---