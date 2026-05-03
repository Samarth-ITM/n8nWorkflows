Automated Client Onboarding System with Notion, Email & CRM Integration

https://n8nworkflows.xyz/workflows/automated-client-onboarding-system-with-notion--email---crm-integration-7781


# Automated Client Onboarding System with Notion, Email & CRM Integration

### 1. Workflow Overview

This workflow, named **"Graceful Client Onboarding Concierge — Pro (General Use, v3)"**, is designed to automate the client onboarding process by integrating data reception, processing, client record creation, notifications, and calendar management. It targets businesses and agencies seeking to streamline new client intake, consent confirmation, and CRM synchronization with minimal manual intervention.

The workflow includes these major logical blocks:

- **1.1 Input Reception:** Handles client intake submissions via webhook or manual trigger simulation.
- **1.2 Intake Data Processing:** Maps and scores the incoming client data to prepare for further steps.
- **1.3 Client Record Creation:** Creates or updates client records in Notion and optionally in HubSpot or Airtable.
- **1.4 Communication & Notification:** Builds and sends welcome emails and sends Telegram notifications to owners if enabled.
- **1.5 Calendar Management:** Optionally creates hold times in Google Calendar for client appointments.
- **1.6 Opt-in Confirmation:** Processes client opt-in confirmations via a separate webhook, updating consent status in Notion.
- **1.7 Error Handling:** Captures workflow errors and notifies the owner via email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block receives incoming client intake data either from a live webhook or a manual trigger for testing purposes. It sets up the user configuration context for downstream processing.

- **Nodes Involved:**  
  - Client Intake Webhook  
  - Manual Trigger  
  - Function: Simulate Intake  
  - Set: User Config

- **Node Details:**

  - **Client Intake Webhook**  
    - Type: Webhook  
    - Role: Entry point for real client intake form submissions.  
    - Configuration: Listens to HTTP requests (method and path not specified here; assumed configured).  
    - Inputs: External HTTP request  
    - Outputs: Passes data downstream to Set: User Config  
    - Edge cases: Invalid or missing payloads, webhook timeout, authentication if applicable.

  - **Manual Trigger**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation for testing the workflow with simulated data.  
    - Configuration: None by default; triggers execution on button press.  
    - Inputs: Manual user action  
    - Outputs: Triggers Function: Simulate Intake  
    - Edge cases: None specific, used for testing.

  - **Function: Simulate Intake**  
    - Type: Function  
    - Role: Generates mock client intake data for testing purposes.  
    - Configuration: Custom JavaScript code to create sample payload mimicking real intake.  
    - Inputs: Manual Trigger  
    - Outputs: Passes simulated data to Function: Map Intake  
    - Edge cases: Simulation data may not cover all real-world cases.

  - **Set: User Config**  
    - Type: Set  
    - Role: Sets user or environment-specific configuration variables needed downstream.  
    - Configuration: Static or dynamic variables configured for the workflow environment (e.g., feature toggles).  
    - Inputs: Client Intake Webhook or Function: Simulate Intake  
    - Outputs: Passes data to Function: Map Intake and Function: Parse Optin  
    - Edge cases: Missing or incorrect config variables may cause downstream failures.

---

#### 2.2 Intake Data Processing

- **Overview:** Maps raw intake data into structured form and scores the lead quality to prioritize client onboarding.

- **Nodes Involved:**  
  - Function: Map Intake  
  - Function: Score Lead

- **Node Details:**

  - **Function: Map Intake**  
    - Type: Function  
    - Role: Transforms raw intake form data into structured client profile data.  
    - Configuration: JavaScript mapping logic aligning input fields to standardized keys (e.g., name, email, preferences).  
    - Inputs: Set: User Config  
    - Outputs: Function: Score Lead  
    - Edge cases: Missing or malformed input fields can cause variable mapping errors.

  - **Function: Score Lead**  
    - Type: Function  
    - Role: Calculates a lead score to assess client priority or suitability.  
    - Configuration: Custom scoring algorithm based on mapped intake data fields (e.g., budget, urgency).  
    - Inputs: Function: Map Intake  
    - Outputs: Notion: Create Client, Function: Build Welcome Email  
    - Edge cases: Scoring logic errors, unexpected data types.

---

#### 2.3 Client Record Creation

- **Overview:** Creates a new client entry in Notion and optionally in external CRM systems based on enabled features.

- **Nodes Involved:**  
  - Notion: Create Client  
  - If: HubSpot Enabled?  
  - HubSpot: Create Contact  
  - If: Airtable Enabled?  
  - Airtable: Create Row

- **Node Details:**

  - **Notion: Create Client**  
    - Type: Notion  
    - Role: Creates a new client page or database entry in Notion to track onboarding and status.  
    - Configuration: Uses Notion API credentials and database IDs; fields mapped from scored lead data.  
    - Inputs: Function: Score Lead  
    - Outputs: Function: Build Welcome Email  
    - Edge cases: API rate limits, invalid credentials, page creation errors.

  - **If: HubSpot Enabled?**  
    - Type: If  
    - Role: Conditional node to check if HubSpot integration is enabled (boolean flag in config).  
    - Configuration: Expression checks config variable.  
    - Inputs: Function: Build Welcome Email  
    - Outputs: HubSpot: Create Contact if true; else no output.  
    - Edge cases: Misconfigured flag causing unwanted API calls.

  - **HubSpot: Create Contact**  
    - Type: HubSpot  
    - Role: Creates a new contact record in HubSpot CRM for the client.  
    - Configuration: Requires HubSpot OAuth2 credentials and contact field mappings.  
    - Inputs: If: HubSpot Enabled? (true branch)  
    - Outputs: None (end of chain)  
    - Edge cases: API errors, duplicate contacts, auth token expiration.

  - **If: Airtable Enabled?**  
    - Type: If  
    - Role: Conditional check for Airtable integration enablement.  
    - Configuration: Expression based on config.  
    - Inputs: Function: Build Welcome Email  
    - Outputs: Airtable: Create Row if true; else no output.  
    - Edge cases: Misconfiguration.

  - **Airtable: Create Row**  
    - Type: Airtable  
    - Role: Adds a new row in Airtable base/table with client data.  
    - Configuration: Airtable API key, base ID, table name, field mappings.  
    - Inputs: If: Airtable Enabled? (true branch)  
    - Outputs: None  
    - Edge cases: API limits, invalid API key, wrong table schema.

---

#### 2.4 Communication & Notification

- **Overview:** Builds and sends a welcome email to the client and optionally notifies the owner via Telegram.

- **Nodes Involved:**  
  - Function: Build Welcome Email  
  - Email: Send Welcome  
  - If: Telegram Enabled?  
  - Telegram: Notify Owner

- **Node Details:**

  - **Function: Build Welcome Email**  
    - Type: Function  
    - Role: Constructs the email body and subject dynamically using client data and templates.  
    - Configuration: JavaScript template strings or similar logic.  
    - Inputs: Notion: Create Client, Function: Score Lead  
    - Outputs: Email: Send Welcome, If: Telegram Enabled?, If: Airtable Enabled?, If: HubSpot Enabled?, If: Calendar Hold Enabled?  
    - Edge cases: Missing client email, template parsing errors.

  - **Email: Send Welcome**  
    - Type: Email Send  
    - Role: Sends the constructed welcome email to the client email address.  
    - Configuration: SMTP or email service credentials (e.g., Outlook OAuth2), email parameters from function output.  
    - Inputs: Function: Build Welcome Email  
    - Outputs: None  
    - Edge cases: SMTP failures, invalid email addresses, network errors.

  - **If: Telegram Enabled?**  
    - Type: If  
    - Role: Checks if Telegram notifications are enabled.  
    - Configuration: Config flag expression.  
    - Inputs: Function: Build Welcome Email  
    - Outputs: Telegram: Notify Owner or none.  
    - Edge cases: Misconfiguration.

  - **Telegram: Notify Owner**  
    - Type: Telegram  
    - Role: Sends notification message to owner/admin via Telegram bot.  
    - Configuration: Telegram bot token and chat ID.  
    - Inputs: If: Telegram Enabled? (true branch)  
    - Outputs: None  
    - Edge cases: Invalid bot token, network issues.

---

#### 2.5 Calendar Management

- **Overview:** Optionally creates hold or reserved times on Google Calendar for the client.

- **Nodes Involved:**  
  - If: Calendar Hold Enabled?  
  - Function: Hold Times  
  - Google Calendar: Create Hold

- **Node Details:**

  - **If: Calendar Hold Enabled?**  
    - Type: If  
    - Role: Checks if calendar hold feature is enabled.  
    - Configuration: Expression from config.  
    - Inputs: Function: Build Welcome Email  
    - Outputs: Function: Hold Times or none.  
    - Edge cases: Config errors.

  - **Function: Hold Times**  
    - Type: Function  
    - Role: Calculates and formats time slots for calendar holds based on client data or config.  
    - Configuration: Custom logic for hold time generation.  
    - Inputs: If: Calendar Hold Enabled?  
    - Outputs: Google Calendar: Create Hold  
    - Edge cases: Timezone issues, invalid date formats.

  - **Google Calendar: Create Hold**  
    - Type: Google Calendar  
    - Role: Creates an event or hold on Google Calendar to reserve client onboarding time.  
    - Configuration: OAuth2 credentials for Google, calendar ID, event details.  
    - Inputs: Function: Hold Times  
    - Outputs: None  
    - Edge cases: API quota exceeded, auth token expiry.

---

#### 2.6 Opt-in Confirmation

- **Overview:** Handles the client opt-in confirmation flow through a dedicated webhook, updating consent and status in Notion.

- **Nodes Involved:**  
  - Opt-in Confirm Webhook  
  - Function: Parse Optin  
  - Notion: Find by Token  
  - Function: Pick Page  
  - Notion: Set Consent + Status

- **Node Details:**

  - **Opt-in Confirm Webhook**  
    - Type: Webhook  
    - Role: Receives client opt-in confirmation requests (likely via a link click).  
    - Configuration: HTTP endpoint listening for confirmation tokens or data.  
    - Inputs: External HTTP request  
    - Outputs: Function: Parse Optin  
    - Edge cases: Invalid or expired tokens, replay attacks.

  - **Function: Parse Optin**  
    - Type: Function  
    - Role: Extracts and validates token or confirmation data from webhook payload.  
    - Configuration: JavaScript parsing logic.  
    - Inputs: Opt-in Confirm Webhook, also connected from Set: User Config for config context.  
    - Outputs: Notion: Find by Token  
    - Edge cases: Missing token, malformed data.

  - **Notion: Find by Token**  
    - Type: Notion  
    - Role: Queries Notion database to find client page by confirmation token.  
    - Configuration: Notion API with filters on token property.  
    - Inputs: Function: Parse Optin  
    - Outputs: Function: Pick Page  
    - Edge cases: No match found, API errors.

  - **Function: Pick Page**  
    - Type: Function  
    - Role: Selects the correct Notion page from search results for update.  
    - Configuration: Logic to pick single page or handle multiple matches.  
    - Inputs: Notion: Find by Token  
    - Outputs: Notion: Set Consent + Status  
    - Edge cases: Multiple or zero pages found.

  - **Notion: Set Consent + Status**  
    - Type: Notion  
    - Role: Updates the consent status and onboarding state for the client page.  
    - Configuration: Notion API update operation with consent fields.  
    - Inputs: Function: Pick Page  
    - Outputs: None  
    - Edge cases: API update failures.

---

#### 2.7 Error Handling

- **Overview:** Captures any errors during workflow execution and notifies the owner by email.

- **Nodes Involved:**  
  - On Error  
  - On Error → Email Owner

- **Node Details:**

  - **On Error**  
    - Type: Error Trigger  
    - Role: Listens for any errors occurring in the workflow runtime.  
    - Configuration: Global error catch configuration.  
    - Inputs: Automatic error trigger  
    - Outputs: On Error → Email Owner  
    - Edge cases: None; serves as safety net.

  - **On Error → Email Owner**  
    - Type: Email Send  
    - Role: Sends an email containing error details to the workflow owner or admin.  
    - Configuration: Email credentials configured, recipient preset.  
    - Inputs: On Error  
    - Outputs: None  
    - Edge cases: Email delivery failure.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                         | Input Node(s)          | Output Node(s)                               | Sticky Note                  |
|--------------------------|---------------------|---------------------------------------|-----------------------|----------------------------------------------|------------------------------|
| Sticky: README           | Sticky Note         | Documentation placeholder             |                       |                                              |                              |
| Sticky: Prereqs + CSV    | Sticky Note         | Documentation placeholder             |                       |                                              |                              |
| Sticky: Setup Checklist  | Sticky Note         | Documentation placeholder             |                       |                                              |                              |
| Sticky: Testing Notes    | Sticky Note         | Documentation placeholder             |                       |                                              |                              |
| Sticky: Compliance + Troubleshooting | Sticky Note | Documentation placeholder             |                       |                                              |                              |
| Sticky: Changelog        | Sticky Note         | Documentation placeholder             |                       |                                              |                              |
| Client Intake Webhook    | Webhook             | Intake data reception endpoint        | External HTTP Request  | Set: User Config                             |                              |
| Manual Trigger           | Manual Trigger      | Manual run and test trigger            | User Action           | Function: Simulate Intake                     |                              |
| Set: User Config         | Set                 | Sets environment and feature flags    | Client Intake Webhook, Function: Simulate Intake | Function: Map Intake, Function: Parse Optin |                              |
| Function: Simulate Intake| Function            | Generates mock intake data for testing| Manual Trigger         | Function: Map Intake                         |                              |
| Function: Map Intake     | Function            | Maps raw intake form data              | Set: User Config       | Function: Score Lead                         |                              |
| Function: Score Lead     | Function            | Scores lead quality                    | Function: Map Intake   | Notion: Create Client, Function: Build Welcome Email |                              |
| Notion: Create Client    | Notion              | Creates client record in Notion        | Function: Score Lead   | Function: Build Welcome Email                |                              |
| Function: Build Welcome Email | Function        | Builds welcome email content           | Notion: Create Client, Function: Score Lead | Email: Send Welcome, If: Telegram Enabled?, If: Airtable Enabled?, If: HubSpot Enabled?, If: Calendar Hold Enabled? |                              |
| Email: Send Welcome      | Email Send          | Sends welcome email to client          | Function: Build Welcome Email |                                            |                              |
| If: Telegram Enabled?    | If                  | Checks if Telegram notifications enabled | Function: Build Welcome Email | Telegram: Notify Owner (true branch)         |                              |
| Telegram: Notify Owner   | Telegram            | Sends notification to owner via Telegram | If: Telegram Enabled? (true branch) |                                            |                              |
| If: Airtable Enabled?    | If                  | Checks if Airtable integration enabled | Function: Build Welcome Email | Airtable: Create Row (true branch)           |                              |
| Airtable: Create Row     | Airtable            | Creates a client record in Airtable    | If: Airtable Enabled? (true branch) |                                            |                              |
| If: HubSpot Enabled?     | If                  | Checks if HubSpot integration enabled  | Function: Build Welcome Email | HubSpot: Create Contact (true branch)         |                              |
| HubSpot: Create Contact  | HubSpot             | Creates contact in HubSpot CRM          | If: HubSpot Enabled? (true branch) |                                            |                              |
| If: Calendar Hold Enabled? | If                | Checks if calendar hold is enabled      | Function: Build Welcome Email | Function: Hold Times (true branch)            |                              |
| Function: Hold Times     | Function            | Calculates hold times for calendar      | If: Calendar Hold Enabled? | Google Calendar: Create Hold                 |                              |
| Google Calendar: Create Hold | Google Calendar | Creates hold event on Google Calendar   | Function: Hold Times   |                                              |                              |
| Opt-in Confirm Webhook   | Webhook             | Receives client opt-in confirmations    | External HTTP Request  | Function: Parse Optin                         |                              |
| Function: Parse Optin    | Function            | Parses and validates opt-in token       | Opt-in Confirm Webhook, Set: User Config | Notion: Find by Token                    |                              |
| Notion: Find by Token    | Notion              | Finds client page by token in Notion    | Function: Parse Optin  | Function: Pick Page                           |                              |
| Function: Pick Page      | Function            | Picks the correct Notion page to update | Notion: Find by Token  | Notion: Set Consent + Status                  |                              |
| Notion: Set Consent + Status | Notion          | Updates consent and status in Notion    | Function: Pick Page    |                                              |                              |
| On Error                 | Error Trigger       | Captures workflow errors                 |                       | On Error → Email Owner                        |                              |
| On Error → Email Owner   | Email Send          | Sends error notification email           | On Error               |                                              |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Client Intake Webhook**  
   - Type: Webhook  
   - Configure HTTP method and path for receiving client intake submissions.  
   - Leave authentication open or configure as needed.

2. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters; used to start manual testing.

3. **Create Function Node: Simulate Intake**  
   - Type: Function  
   - JavaScript code to generate mock client intake data resembling the real payload structure.

4. **Create Set Node: User Config**  
   - Type: Set  
   - Define configuration variables such as feature toggles for integrations (Telegram, Airtable, HubSpot, Calendar hold).  
   - Set any necessary environment variables or static user data.

5. **Connect Client Intake Webhook and Function: Simulate Intake to Set: User Config**  
   - Both nodes output to Set: User Config (parallel paths merged).

6. **Create Function Node: Map Intake**  
   - Type: Function  
   - Code to map raw intake fields to standard client data keys.

7. **Connect Set: User Config to Function: Map Intake**

8. **Create Function Node: Score Lead**  
   - Type: Function  
   - Logic to score or prioritize the lead based on input data.

9. **Connect Function: Map Intake to Function: Score Lead**

10. **Create Notion Node: Create Client**  
    - Type: Notion  
    - Configure with Notion API credentials and target database/page.  
    - Map scored lead data to Notion fields.

11. **Connect Function: Score Lead to Notion: Create Client**

12. **Create Function Node: Build Welcome Email**  
    - Type: Function  
    - Build email subject and body using client data.

13. **Connect Notion: Create Client and Function: Score Lead to Function: Build Welcome Email**

14. **Create Email Send Node: Email: Send Welcome**  
    - Type: Email Send  
    - Configure SMTP or OAuth2 credentials (e.g., Outlook OAuth2).  
    - Use email content from previous node.

15. **Connect Function: Build Welcome Email to Email: Send Welcome**

16. **Create If Node: If Telegram Enabled?**  
    - Type: If  
    - Expression checks boolean config flag for Telegram integration.

17. **Create Telegram Node: Notify Owner**  
    - Type: Telegram  
    - Configure Telegram bot token and chat ID.

18. **Connect Function: Build Welcome Email to If: Telegram Enabled?**  
    - Connect true branch to Telegram: Notify Owner

19. **Create If Node: If Airtable Enabled?**  
    - Type: If  
    - Expression checks Airtable integration flag.

20. **Create Airtable Node: Create Row**  
    - Type: Airtable  
    - Configure Airtable API key, base ID, table name, and field mappings.

21. **Connect Function: Build Welcome Email to If: Airtable Enabled?**  
    - Connect true branch to Airtable: Create Row

22. **Create If Node: If HubSpot Enabled?**  
    - Type: If  
    - Expression checks HubSpot integration flag.

23. **Create HubSpot Node: Create Contact**  
    - Type: HubSpot  
    - Configure HubSpot OAuth2 credentials and contact field mappings.

24. **Connect Function: Build Welcome Email to If: HubSpot Enabled?**  
    - Connect true branch to HubSpot: Create Contact

25. **Create If Node: If Calendar Hold Enabled?**  
    - Type: If  
    - Expression checks calendar hold feature flag.

26. **Create Function Node: Hold Times**  
    - Type: Function  
    - Logic to calculate calendar hold time slots.

27. **Create Google Calendar Node: Create Hold**  
    - Type: Google Calendar  
    - Configure Google OAuth2 credentials, calendar ID, and event details.

28. **Connect Function: Build Welcome Email to If: Calendar Hold Enabled?**  
    - Connect true branch to Function: Hold Times  
    - Connect Function: Hold Times to Google Calendar: Create Hold

29. **Create Webhook Node: Opt-in Confirm Webhook**  
    - Type: Webhook  
    - Configure HTTP method and path to receive opt-in confirmation requests.

30. **Create Function Node: Parse Optin**  
    - Type: Function  
    - Parse and validate opt-in tokens from webhook payload.

31. **Connect Opt-in Confirm Webhook to Function: Parse Optin**

32. **Connect Set: User Config to Function: Parse Optin**  
    - For configuration context.

33. **Create Notion Node: Find by Token**  
    - Type: Notion  
    - Configure query to find client page by token attribute.

34. **Connect Function: Parse Optin to Notion: Find by Token**

35. **Create Function Node: Pick Page**  
    - Type: Function  
    - Logic to select the correct Notion page from search results.

36. **Connect Notion: Find by Token to Function: Pick Page**

37. **Create Notion Node: Set Consent + Status**  
    - Type: Notion  
    - Configure update operation to set consent and onboarding status fields.

38. **Connect Function: Pick Page to Notion: Set Consent + Status**

39. **Create Error Trigger Node: On Error**  
    - Type: Error Trigger  
    - No parameters; listens for any workflow errors.

40. **Create Email Send Node: On Error → Email Owner**  
    - Type: Email Send  
    - Configure email credentials and recipient address for error notifications.

41. **Connect On Error to On Error → Email Owner**

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow requires attaching API credentials securely after import: Notion, HubSpot, Airtable, Google OAuth2 for Calendar, SMTP or Outlook OAuth2 for Email, Telegram Bot Token. | Credential setup instructions.                   |
| Testing can be done via the Manual Trigger node which simulates intake data for end-to-end validation.       | Testing notes.                                   |
| Error handling notifies the owner by email, ensuring operational awareness of any failures.                  | Operational monitoring.                          |
| For configuration, feature toggles enable or disable integrations dynamically without editing node connections.| Configuration flexibility.                       |
| Workflow designed for modular extension; new integrations can be added by following the existing pattern.    | Scalability and extensibility guidance.         |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.