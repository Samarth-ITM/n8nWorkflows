HubSpot Contact Email Validation with Hunter.io

https://n8nworkflows.xyz/workflows/hubspot-contact-email-validation-with-hunter-io-4416


# HubSpot Contact Email Validation with Hunter.io

### 1. Workflow Overview

This workflow automates the validation of email addresses of HubSpot contacts using the Hunter.io email verification service. It systematically fetches contacts from HubSpot who have an email but do not yet have a Hunter verification status, verifies each email via Hunter.io, updates the contact record in HubSpot with validation results, and finally sends a notification email upon completion.

The workflow is logically organized into these functional blocks:

- **1.1 Input Trigger and Contact Retrieval:** Manual trigger to start the process, followed by fetching the relevant contacts from HubSpot.
- **1.2 Batch Processing Loop:** Splits the list of contacts into manageable batches to process each contact individually.
- **1.3 Email Verification via Hunter.io:** For each contact, verifies the email address using Hunter's Email Verifier API.
- **1.4 Updating HubSpot Contacts:** Updates the HubSpot contact with verification status and date.
- **1.5 Rate Limiting Wait:** Introduces a wait time between API calls to avoid hitting rate limits.
- **1.6 Workflow Completion Notification:** Sends an email notifying that the verification process is complete.
- **1.7 Informational Sticky Notes:** Provide customization guidance and explanations for workflow users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and Contact Retrieval

- **Overview:** This block starts the workflow manually and retrieves all HubSpot contacts that have an email but have not yet been verified by Hunter.io.
- **Nodes Involved:** 
  - When clicking ‘Test workflow’
  - HubSpot
- **Node Details:**

  - **When clicking ‘Test workflow’**
    - Type: Manual Trigger
    - Role: Starts the workflow manually.
    - Configuration: Default manual trigger; no parameters.
    - Input: None (trigger node).
    - Output: Starts the workflow chain.
    - Potential Failures: None typical; user must manually trigger.

  - **HubSpot**
    - Type: HubSpot node
    - Role: Searches HubSpot contacts with email and without Hunter verification status.
    - Configuration:
      - Operation: Search contacts
      - Filter: Contacts must have ‘email’ property and must NOT have ‘hunter_email_validation_status’ property.
      - Return All: True (fetch all matching contacts)
      - Authentication: App Token (HubSpot App Token credential)
    - Key Expressions: Uses HubSpot filter groups to filter contacts.
    - Input: Trigger from manual node.
    - Output: List of contacts matching filter.
    - Failure Modes:
      - Authentication errors if token invalid.
      - API rate limits or timeouts.
      - No contacts found leads to empty output but workflow continues.

#### 1.2 Batch Processing Loop

- **Overview:** Splits the contact list into batches and iterates over each contact individually to allow sequential processing.
- **Nodes Involved:**
  - Loop Over Items
- **Node Details:**

  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Processes contacts one at a time or in batches.
    - Configuration:
      - Reset option disabled (does not restart batches).
      - Default batch size (implicitly 1, as no batch size specified).
    - Input: List of HubSpot contacts.
    - Output: Single contact item per iteration.
    - Failure Modes:
      - If input is empty, no iterations occur.
      - Possible misconfiguration if batch size is not set (default is 1).
  
#### 1.3 Email Verification via Hunter.io

- **Overview:** Verifies each contact’s email address using Hunter.io’s Email Verifier API.
- **Nodes Involved:**
  - Hunter
- **Node Details:**

  - **Hunter**
    - Type: Hunter node
    - Role: Calls Hunter.io Email Verifier to validate the email.
    - Configuration:
      - Operation: emailVerifier
      - Email: Extracted from current contact’s email property (`{{$json.properties.email}}`).
      - Authentication: Hunter API credentials.
    - Input: Single contact from Loop Over Items.
    - Output: Verification result JSON.
    - Failure Modes:
      - API rate limit exceeded.
      - Invalid or missing email address.
      - Authentication failure.
      - Network timeouts.

#### 1.4 Updating HubSpot Contacts

- **Overview:** Updates HubSpot contact records with Hunter verification results and verification date.
- **Nodes Involved:**
  - Add Hunter Details (Contact)
- **Node Details:**

  - **Add Hunter Details (Contact)**
    - Type: HTTP Request
    - Role: PATCH update to HubSpot contact properties.
    - Configuration:
      - URL: HubSpot API endpoint for the specific contact using ID from Loop Over Items.
      - Method: PATCH
      - Headers: Content-Type application/json.
      - Body (JSON):
        - Sets `hunter_email_validation_status` with Hunter’s status.
        - Sets `hunter_verification_date` with current date formatted as yyyy-MM-dd.
      - Authentication: HubSpot App Token credential.
    - Input: Verification result from Hunter and contact ID from Loop Over Items.
    - Output: Response from HubSpot API, confirming update.
    - Failure Modes:
      - Invalid contact ID.
      - Authentication errors.
      - API rate limits.
      - JSON body formatting errors.
  
#### 1.5 Rate Limiting Wait

- **Overview:** A wait node pauses workflow execution for 1 second between updates to avoid Hunter.io and HubSpot API rate limits.
- **Nodes Involved:**
  - Wait
- **Node Details:**

  - **Wait**
    - Type: Wait node
    - Role: Delay execution by specified time.
    - Configuration:
      - Amount: 1 second.
    - Input: Output from Add Hunter Details (Contact).
    - Output: After 1-second delay, passes control to next node.
    - Failure Modes:
      - None typical; if workflow paused or system sleeps, delay may be longer.
  
#### 1.6 Workflow Completion Notification

- **Overview:** Sends a notification email upon completion of all email verifications.
- **Nodes Involved:**
  - Send Email
- **Node Details:**

  - **Send Email**
    - Type: Email Send node
    - Role: Sends an email to notify that verification is complete.
    - Configuration:
      - HTML body with styled message confirming completion.
      - Subject: "Email Verification Completed for Your HubSpot Contacts"
      - To Email: Fixed recipient email (akhilgadiraju@gmail.com).
      - From Email: Fixed sender email (test@gmail.com).
      - SMTP Credentials configured.
      - Append Attribution disabled.
    - Input: After Loop Over Items completes all iterations.
    - Output: Email sent confirmation.
    - Failure Modes:
      - SMTP authentication errors.
      - Invalid recipient email.
      - Email formatting issues.

#### 1.7 Informational Sticky Notes

- **Overview:** Provides user guidance on customization and workflow behavior.
- **Nodes Involved:**
  - Sticky Note
  - Sticky Note1
  - Sticky Note2
- **Node Details:**

  - **Sticky Note**
    - Content: "⚠️ Customize Fields - Rename the field names to match your preference"
    - Positioned near Hunter and Add Hunter Details nodes.

  - **Sticky Note1**
    - Content: "🕛 Wait - This node is responsible for halting the workflow for one second to avoid rate limit limitations."
    - Positioned near Wait node.

  - **Sticky Note2**
    - Content: "⚠️ Customize Fields - you can use any field name that suits your preference, make sure you add those properties in HubSpot first."
    - Positioned near HubSpot node.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                           | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                   |
|----------------------------|---------------------|-----------------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually             | -                           | HubSpot                      |                                                                                              |
| HubSpot                    | HubSpot             | Searches contacts with email and without Hunter status | When clicking ‘Test workflow’ | Loop Over Items              | ⚠️ Customize Fields - you can use any field name that suits your preference, make sure you add those properties in hubspot first. |
| Loop Over Items            | SplitInBatches      | Processes contacts one at a time         | HubSpot                     | Send Email, Hunter            |                                                                                              |
| Hunter                     | Hunter              | Verifies each contact's email            | Loop Over Items             | Add Hunter Details (Contact)  | ⚠️ Customize Fields - Rename the field names to match your preference                         |
| Add Hunter Details (Contact) | HTTP Request        | Updates HubSpot contact with verification status | Hunter                      | Wait                         | ⚠️ Customize Fields - Rename the field names to match your preference                         |
| Wait                       | Wait                | Delays workflow to avoid API rate limits | Add Hunter Details (Contact) | Replace Me                   | 🕛 Wait - This node is responsible for halting the workflow for one second to avoid rate limit limitatins. |
| Replace Me                 | NoOp                | Placeholder/no operation node             | Wait                        | Loop Over Items              |                                                                                              |
| Send Email                 | Email Send          | Sends notification email after completion | Loop Over Items             | -                            |                                                                                              |
| Sticky Note                | Sticky Note         | Guidance on customizing field names      | -                           | -                            | ⚠️ Customize Fields - Rename the field names to match your preference                         |
| Sticky Note1               | Sticky Note         | Explanation of the Wait node's purpose   | -                           | -                            | 🕛 Wait - This node is responsible for halting the workflow for one second to avoid rate limit limitatins. |
| Sticky Note2               | Sticky Note         | Guidance on HubSpot property customization | -                           | -                            | ⚠️ Customize Fields - you can use any field name that suits your preference, make sure you add those properties in hubspot first. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.
   - No parameters needed.

2. **Add HubSpot Node to Search Contacts**
   - Add **HubSpot** node named `HubSpot`.
   - Set Operation to `search`.
   - Filter criteria:
     - `email` property `HAS_PROPERTY`.
     - `hunter_email_validation_status` property `NOT_HAS_PROPERTY`.
   - Set `Return All` to `true`.
   - Set Authentication to use your **HubSpot App Token** credentials.
   - Connect `When clicking ‘Test workflow’` → `HubSpot`.

3. **Add SplitInBatches Node to Loop Over Contacts**
   - Add **SplitInBatches** node named `Loop Over Items`.
   - Default batch size is 1 (process contacts one by one).
   - Connect `HubSpot` → `Loop Over Items`.

4. **Add Hunter Node for Email Verification**
   - Add **Hunter** node named `Hunter`.
   - Operation: `emailVerifier`.
   - Email parameter: Use expression `={{ $json.properties.email }}` to extract current contact’s email.
   - Set Hunter API credentials.
   - Connect `Loop Over Items` → `Hunter`.

5. **Add HTTP Request Node to Update HubSpot Contact**
   - Add **HTTP Request** node named `Add Hunter Details (Contact)`.
   - Method: `PATCH`.
   - URL: `https://api.hubapi.com/crm/v3/objects/contacts/{{ $('Loop Over Items').item.json.id }}` (use expression to get contact ID).
   - Headers: `Content-Type: application/json`.
   - Body (JSON):
     ```json
     {
       "properties": {
         "hunter_email_validation_status": "{{ $json.status }}",
         "hunter_verification_date": "{{ $now.format('yyyy-MM-dd') }}"
       }
     }
     ```
   - Authentication: Use your **HubSpot App Token** credentials.
   - Connect `Hunter` → `Add Hunter Details (Contact)`.

6. **Add Wait Node to Prevent Rate Limiting**
   - Add **Wait** node named `Wait`.
   - Set amount to 1 second.
   - Connect `Add Hunter Details (Contact)` → `Wait`.

7. **Add NoOp Node as Loop Continuation Placeholder**
   - Add **NoOp** node named `Replace Me`.
   - Connect `Wait` → `Replace Me`.
   - Connect `Replace Me` → `Loop Over Items` (to continue looping).

8. **Add Email Send Node for Completion Notification**
   - Add **Email Send** node named `Send Email`.
   - Configure SMTP credentials.
   - Set `To Email` to the desired recipient.
   - Set `From Email` as appropriate.
   - Subject: "Email Verification Completed for Your HubSpot Contacts".
   - HTML Body: Use the provided styled HTML content or customize.
   - Connect `Loop Over Items` main output (after all batches processed) → `Send Email`.

9. **Add Sticky Notes for Documentation**
   - Add Sticky Notes near:
     - HubSpot and Hunter nodes about customizing property names.
     - Wait node describing its role in avoiding rate limits.

10. **Credentials Setup**
    - HubSpot App Token for HubSpot nodes and HTTP Requests.
    - Hunter API key for Hunter node.
    - SMTP credentials for Email Send node.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                   |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------|
| ⚠️ Customize the HubSpot property field names in the workflow to match your CRM schema.       | Important for accurate data mapping.              |
| 🕛 The Wait node pauses the workflow for 1 second to prevent API rate limit issues.            | Critical for avoiding throttling by Hunter.io and HubSpot APIs. |
| Email notification sent to a fixed address upon workflow completion; update as needed.        | Allows alerting stakeholders of process completion. |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.