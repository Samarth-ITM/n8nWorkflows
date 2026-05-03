Automated Email Optin Form with n8n and Hunter io for verification

https://n8nworkflows.xyz/workflows/automated-email-optin-form-with-n8n-and-hunter-io-for-verification-2709


# Automated Email Optin Form with n8n and Hunter io for verification

### 1. Workflow Overview

This workflow automates the collection and verification of email subscribers using n8n, Hunter.io, and SendGrid. It is designed for users who want a cost-effective, scalable email opt-in form with built-in email validation to ensure only legitimate addresses are collected. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures email submissions from a customizable embedded form.
- **1.2 Email Verification:** Uses Hunter.io to verify the validity of submitted email addresses.
- **1.3 Conditional Routing:** Checks verification results to decide further processing.
- **1.4 Contact Addition:** Adds verified emails to a SendGrid contact list.
- **1.5 Handling Invalid Emails:** Gracefully ignores invalid email submissions without further action.
- **1.6 Informational Notes:** Provides user guidance and setup instructions via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures email addresses submitted via a web form embedded on a website. It initiates the workflow when a user submits their email.

- **Nodes Involved:**  
  - Submit form

- **Node Details:**  
  - **Submit form**  
    - Type: Form Trigger  
    - Role: Entry point for email submissions  
    - Configuration:  
      - Webhook ID configured for external form embedding  
      - Form titled "Join my mailing list now"  
      - Single required field labeled "Email"  
      - Form description promotes AI productivity tips  
    - Inputs: External HTTP POST from embedded form  
    - Outputs: JSON containing submitted email under the key `Email`  
    - Edge Cases:  
      - Missing or malformed email field (form requires email, so unlikely)  
      - Webhook connectivity issues or unauthorized access  
    - Version: 2.2

#### 2.2 Email Verification

- **Overview:**  
  This block verifies the submitted email address using Hunter.io’s email verification API to filter out invalid or fake emails.

- **Nodes Involved:**  
  - Verify email

- **Node Details:**  
  - **Verify email**  
    - Type: Hunter node (Email Verifier)  
    - Role: Calls Hunter.io API to verify email validity  
    - Configuration:  
      - Email parameter dynamically set from form submission (`{{$json.Email}}`)  
      - Operation: Email verification  
      - Credentials: Hunter.io API key configured (requires user setup)  
    - Inputs: Email from Submit form node  
    - Outputs: JSON including verification status (e.g., "valid", "invalid")  
    - Edge Cases:  
      - API quota exceeded (Hunter.io free tier limit: 50/month)  
      - API key invalid or expired  
      - Network timeouts or API errors  
    - Version: 1

#### 2.3 Conditional Routing

- **Overview:**  
  This block evaluates the verification result and routes the workflow accordingly: valid emails proceed to contact addition; invalid emails are ignored.

- **Nodes Involved:**  
  - Check if the email is valid

- **Node Details:**  
  - **Check if the email is valid**  
    - Type: If node  
    - Role: Conditional branching based on Hunter.io verification status  
    - Configuration:  
      - Condition: Checks if `{{$json.status}}` equals `"valid"` (case-sensitive, strict)  
    - Inputs: Output from Verify email node  
    - Outputs:  
      - True branch: For valid emails  
      - False branch: For invalid emails  
    - Edge Cases:  
      - Missing or malformed `status` field in API response  
      - Unexpected status values  
    - Version: 2

#### 2.4 Contact Addition

- **Overview:**  
  This block adds verified email addresses to a SendGrid contact list for email marketing purposes.

- **Nodes Involved:**  
  - Add contact to list

- **Node Details:**  
  - **Add contact to list**  
    - Type: SendGrid node  
    - Role: Adds a contact to a specified SendGrid list  
    - Configuration:  
      - Email dynamically set from input (`{{$json.Email}}`)  
      - Resource: Contact  
      - Additional fields: List ID set to `"11a55438-d4a8-4740-b054-d273359b7dfe"` (predefined SendGrid list)  
      - Credentials: SendGrid API key configured (requires user setup)  
    - Inputs: True branch from Check if the email is valid node  
    - Outputs: Confirmation of contact addition  
    - Edge Cases:  
      - Invalid or expired SendGrid API key  
      - List ID does not exist or is inaccessible  
      - API rate limits or network errors  
    - Version: 1

#### 2.5 Handling Invalid Emails

- **Overview:**  
  This block handles invalid email submissions by performing no further action, effectively ignoring them.

- **Nodes Involved:**  
  - Email is not valid, do nothing

- **Node Details:**  
  - **Email is not valid, do nothing**  
    - Type: No Operation (NoOp) node  
    - Role: Terminates workflow branch for invalid emails without side effects  
    - Configuration: No parameters  
    - Inputs: False branch from Check if the email is valid node  
    - Outputs: None  
    - Edge Cases: None (safe no-op)  
    - Version: 1

#### 2.6 Informational Notes

- **Overview:**  
  These nodes provide contextual information and setup instructions for users maintaining or deploying the workflow.

- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides links to case study and tutorial video  
    - Content:  
      - Link to detailed blog post guide  
      - Link to YouTube tutorial video  
    - Position: Top-left for visibility  
  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Reminds user to set up Hunter.io account and API key  
    - Content: Notes 50 free monthly credits on Hunter.io  
    - Positioned near Verify email node for context

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                      | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                           |
|-------------------------------|---------------------|------------------------------------|------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Submit form                   | Form Trigger        | Captures email submissions         | (Webhook external)      | Verify email                |                                                                                                     |
| Verify email                 | Hunter              | Verifies email validity via API    | Submit form            | Check if the email is valid |                                                                                                     |
| Check if the email is valid  | If                  | Routes based on email validity     | Verify email           | Add contact to list, Email is not valid, do nothing |                                                                                                     |
| Add contact to list          | SendGrid            | Adds verified email to SendGrid list | Check if the email is valid (true branch) | None                        |                                                                                                     |
| Email is not valid, do nothing | NoOp                | Terminates invalid email branch    | Check if the email is valid (false branch) | None                        |                                                                                                     |
| Sticky Note1                 | Sticky Note         | Provides case study and tutorial links | None                   | None                        | ## Automate Email List Building with n8n and Hunter io\n\n💡 Read the [case study here](https://rumjahn.com/create-email-capture-forms-for-free-using-n8n-and-sendgrid-and-easily-grow-your-subscriber-list/).\n\n📺 Watch the [youtube tutorial here](https://www.youtube.com/watch?v=NgvEHwu19Rs&t=2s) |
| Sticky Note                  | Sticky Note         | Hunter.io setup reminder            | None                   | None                        | ## Hunter io\n\nYou need to get a Hunter.io account and input the API key. There's 50 free credits per month. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named `Submit form`.  
   - Configure the webhook with a unique webhook ID (auto-generated or custom).  
   - Set the form title to "Join my mailing list now".  
   - Add one form field:  
     - Label: "Email"  
     - Required: Yes  
   - Add a form description: "10x your productivity with my A.I. tips. I'll cut the B.S. and give you the most practical tips for A.I. automation."  
   - Save the node.

2. **Add Hunter.io Email Verification Node**  
   - Add a **Hunter** node named `Verify email`.  
   - Set operation to "emailVerifier".  
   - Set the email parameter to `={{ $json.Email }}` to dynamically use the submitted email.  
   - Configure Hunter.io credentials:  
     - Create or select existing Hunter.io API credentials with a valid API key.  
   - Connect the output of `Submit form` to the input of `Verify email`.

3. **Add Conditional Check Node**  
   - Add an **If** node named `Check if the email is valid`.  
   - Configure the condition:  
     - Check if the expression `{{$json.status}}` equals the string `"valid"` (case-sensitive).  
   - Connect the output of `Verify email` to the input of this node.

4. **Add SendGrid Contact Addition Node**  
   - Add a **SendGrid** node named `Add contact to list`.  
   - Set resource to "contact".  
   - Set the email parameter to `={{ $json.Email }}`.  
   - Under additional fields, set the list ID to your SendGrid list ID (e.g., `"11a55438-d4a8-4740-b054-d273359b7dfe"`).  
   - Configure SendGrid credentials:  
     - Create or select existing SendGrid API credentials with a valid API key.  
   - Connect the **true** output of `Check if the email is valid` to this node.

5. **Add No Operation Node for Invalid Emails**  
   - Add a **NoOp** node named `Email is not valid, do nothing`.  
   - No configuration needed.  
   - Connect the **false** output of `Check if the email is valid` to this node.

6. **Add Sticky Notes for Documentation (Optional)**  
   - Add a **Sticky Note** node named `Sticky Note1`.  
   - Paste content with links to the case study and YouTube tutorial:  
     ```
     ## Automate Email List Building with n8n and Hunter io

     💡 Read the [case study here](https://rumjahn.com/create-email-capture-forms-for-free-using-n8n-and-sendgrid-and-easily-grow-your-subscriber-list/).

     📺 Watch the [youtube tutorial here](https://www.youtube.com/watch?v=NgvEHwu19Rs&t=2s)
     ```  
   - Add another **Sticky Note** node named `Sticky Note`.  
   - Paste content reminding about Hunter.io setup and free credits:  
     ```
     ## Hunter io

     You need to get a Hunter.io account and input the API key. There's 50 free credits per month.
     ```  
   - Position these notes near relevant nodes for clarity.

7. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Embed the form on your website using the webhook URL from the `Submit form` node.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is built by [rumjahn](https://rumjahn.com) and designed for free email list building. | Author credit and project origin.                                                                   |
| Watch the tutorial video to understand setup and customization: https://www.youtube.com/watch?v=NgvEHwu19Rs&t=2s | Video tutorial for visual guidance.                                                                |
| Read the detailed blog post for step-by-step instructions and use cases: https://rumjahn.com/create-email-capture-forms-for-free-using-n8n-and-sendgrid-and-easily-grow-your-subscriber-list/ | Comprehensive written guide with examples and tips.                                                |
| Hunter.io provides 50 free email verification credits monthly; consider account limits when scaling. | Important for managing API usage and avoiding service interruptions.                                |
| Customize the SendGrid list ID to match your own contact list for proper integration.             | Ensures emails are added to the correct marketing list.                                            |
| The form trigger node supports embedding on any website via webhook URL, enabling flexible deployment. | Facilitates easy integration with existing websites or landing pages.                              |

---

This document fully describes the workflow’s structure, logic, and setup, enabling advanced users and automation agents to understand, reproduce, and customize the email opt-in and verification process efficiently.