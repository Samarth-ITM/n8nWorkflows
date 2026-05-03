🔐 Double Opt-In Email Verification System with Google Sheets

https://n8nworkflows.xyz/workflows/---double-opt-in-email-verification-system-with-google-sheets-3953


# 🔐 Double Opt-In Email Verification System with Google Sheets

---

## 1. Workflow Overview

This workflow implements a **Double Opt-In Email Verification System** using **Google Sheets** as a backend database and SMTP for email delivery. It is designed to securely verify user email addresses before granting access to a main form or process, ensuring GDPR compliance, reducing spam, and improving data quality.

### Logical Blocks:

- **1.1 Input Reception and Initial Email Capture**  
  Captures the user’s initial email submission via a form trigger and a linked email input form.

- **1.2 Verification Code Generation and Storage**  
  Generates a unique 6-digit verification code, stores it along with user data in Google Sheets.

- **1.3 Email Dispatch of Verification Code**  
  Sends a verification email containing the unique code to the user.

- **1.4 Code Verification Input and Validation**  
  Presents a verification form to the user to enter the code, then checks the validity of the entered code.

- **1.5 Handling Verification Outcomes**  
  If the code is valid, grants access to the main form; if invalid, allows a retry or reset process with error messages.

- **1.6 Main Form Access and Continuation**  
  Upon successful verification, user proceeds to the main application form or process.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Initial Email Capture

- **Overview:**  
  This block initiates the workflow by capturing the user’s initial email submission via a form trigger, followed by presenting an email input form for confirmation or additional data.

- **Nodes Involved:**  
  - On form submission  
  - Email Form  

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger (Webhook-based)  
    - Role: Listens for the initial form submission event to start the workflow.  
    - Configuration: Default, listens on a webhook for form submission.  
    - Inputs: External form submission trigger.  
    - Outputs: Passes form data to "Email Form" node.  
    - Edge Cases: Possible webhook unavailability or malformed submissions.

  - **Email Form**  
    - Type: Form Node  
    - Role: Presents a form to capture or confirm the user's email address.  
    - Configuration: Customizable form fields for email input; linked to a webhook.  
    - Inputs: Triggered from "On form submission".  
    - Outputs: Sends captured email to "Generate Code" node.  
    - Edge Cases: Invalid email formats, empty submissions.

---

### 2.2 Verification Code Generation and Storage

- **Overview:**  
  Generates a unique 6-digit verification code for the user and securely stores the email, code, and related data in Google Sheets.

- **Nodes Involved:**  
  - Generate Code  
  - Store Data  

- **Node Details:**

  - **Generate Code**  
    - Type: Code Node (JavaScript)  
    - Role: Creates a unique 6-digit numeric verification code.  
    - Configuration: Contains JavaScript code generating a zero-padded 6-digit string.  
    - Inputs: Receives user email data from "Email Form".  
    - Outputs: Adds generated code to data object, forwards to "Store Data".  
    - Edge Cases: Code generation failure (very rare), data type mismatches.

  - **Store Data**  
    - Type: Google Sheets Node  
    - Role: Writes user data and verification code to a configured Google Sheet.  
    - Configuration:  
      - Operation: Append or Update Row  
      - Sheet ID: Must be replaced with the user's Google Sheets ID  
      - Sheet Tab: Correct tab selected with columns for Start Date, ID, Accepts Terms, Email, Code  
    - Inputs: Receives data including email and generated code.  
    - Outputs: Passes data to "Send Email".  
    - Edge Cases: Authentication errors, API rate limits, incorrect sheet ID or permissions, network issues.

---

### 2.3 Email Dispatch of Verification Code

- **Overview:**  
  Sends an email containing the verification code to the user’s email address using SMTP credentials.

- **Nodes Involved:**  
  - Send Email  

- **Node Details:**

  - **Send Email**  
    - Type: Email Send Node (SMTP)  
    - Role: Sends the verification email with the unique code embedded in the message.  
    - Configuration:  
      - SMTP credentials must be configured in n8n (host, port, user, password)  
      - From Email: e.g., `no-reply@yourdomain.com`  
      - To: User's email from previous node  
      - Subject and message body: Configurable with placeholders for code and branding  
      - Reply-To: Support email address  
    - Inputs: Receives user email and code from "Store Data".  
    - Outputs: Triggers "Verification Form".  
    - Edge Cases: SMTP authentication failure, email delivery issues, invalid email addresses, spam filtering.

---

### 2.4 Code Verification Input and Validation

- **Overview:**  
  Presents a form for the user to enter the received verification code and validates it against stored data.

- **Nodes Involved:**  
  - Verification Form  
  - Check Code  

- **Node Details:**

  - **Verification Form**  
    - Type: Form Node  
    - Role: Allows the user to input the verification code received by email.  
    - Configuration: Form with input field for the code, linked via webhook.  
    - Inputs: Triggered after "Send Email".  
    - Outputs: Passes user input to "Check Code".  
    - Edge Cases: Empty submissions, invalid code formats.

  - **Check Code**  
    - Type: If Node (Conditional)  
    - Role: Compares entered code with stored code in Google Sheets.  
    - Configuration: Uses expressions to compare input code with stored value.  
    - Inputs: Receives verification form input.  
    - Outputs:  
      - If valid: proceeds to "Main Form" access.  
      - If invalid: proceeds to "Incorrect Code Form".  
    - Edge Cases: Code mismatch, expired codes if implemented, multiple attempts.

---

### 2.5 Handling Verification Outcomes

- **Overview:**  
  Manages responses to verification success or failure, allowing retries or workflow resets after multiple failed attempts.

- **Nodes Involved:**  
  - Incorrect Code Form  
  - Second Check  
  - Reset Form  
  - Continue With Your Flow (NoOp)  

- **Node Details:**

  - **Incorrect Code Form**  
    - Type: Form Node  
    - Role: Displays an error message and allows the user to retry entering the code.  
    - Inputs: From "Check Code" when code is invalid.  
    - Outputs: Passes data to "Second Check".  
    - Edge Cases: User abandonment, repeated failures.

  - **Second Check**  
    - Type: If Node (Conditional)  
    - Role: Evaluates if the user has exceeded allowed retry attempts.  
    - Inputs: Receives retry attempt count or flags from "Incorrect Code Form".  
    - Outputs:  
      - If retry attempts remain: loops back to "Main Form".  
      - If attempts exceeded: triggers "Reset Form".  
    - Edge Cases: Incorrect retry counting, infinite loops.

  - **Reset Form**  
    - Type: Form Node  
    - Role: Provides option to restart the entire verification process.  
    - Inputs: Triggered after exceeding attempts in "Second Check".  
    - Outputs: Redirects back to "Email Form" for a fresh start.  
    - Edge Cases: User drop-off, data cleanup considerations.

  - **Continue With Your Flow**  
    - Type: No Operation (NoOp) Node  
    - Role: Placeholder to continue the main application logic after successful verification.  
    - Inputs: From "Main Form".  
    - Outputs: None in this workflow; meant for user customization.  
    - Edge Cases: None.

---

### 2.6 Main Form Access and Continuation

- **Overview:**  
  Grants the verified user access to the main form or process after successful email confirmation.

- **Nodes Involved:**  
  - Main Form  

- **Node Details:**

  - **Main Form**  
    - Type: Form Node  
    - Role: Presents the main form or application process to the verified user.  
    - Configuration: Customizable form fields as per user’s needs.  
    - Inputs: Triggered from "Check Code" on success or "Second Check" retry.  
    - Outputs: Passes control to "Continue With Your Flow".  
    - Edge Cases: Form submission errors, unauthorized access if bypassed.

---

## 3. Summary Table

| Node Name           | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note              |
|---------------------|---------------------|---------------------------------------|------------------------|--------------------------|--------------------------|
| On form submission  | Form Trigger        | Start workflow on initial form submit | None                   | Email Form               |                          |
| Email Form          | Form                | Capture user email                     | On form submission     | Generate Code            |                          |
| Generate Code       | Code                | Generate unique 6-digit code           | Email Form             | Store Data               |                          |
| Store Data          | Google Sheets       | Store email and code in Google Sheets | Generate Code          | Send Email               |                          |
| Send Email          | Email Send (SMTP)   | Send verification code via email       | Store Data             | Verification Form        |                          |
| Verification Form   | Form                | Input for user to enter verification code | Send Email           | Check Code               |                          |
| Check Code          | If                  | Validate entered code against stored   | Verification Form      | Main Form, Incorrect Code Form |                    |
| Incorrect Code Form | Form                | Show error message on invalid code     | Check Code (false)     | Second Check             |                          |
| Second Check        | If                  | Check retry attempts                    | Incorrect Code Form    | Main Form, Reset Form    |                          |
| Reset Form          | Form                | Allow restart after too many failures  | Second Check (false)   | Email Form               |                          |
| Main Form           | Form                | Main application form for verified user | Check Code (true), Second Check (true) | Continue With Your Flow |                          |
| Continue With Your Flow | NoOp              | Placeholder for next workflow steps     | Main Form              | None                    |                          |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Name: `On form submission`  
   - Type: Form Trigger (Webhook)  
   - Purpose: Listen for initial form submissions to start the workflow.

2. **Create Email Form Node:**  
   - Name: `Email Form`  
   - Type: Form  
   - Configure fields to capture user email (and optionally terms acceptance).  
   - Connect output from `On form submission` to this node.

3. **Add Code Node for Verification Code:**  
   - Name: `Generate Code`  
   - Type: Code  
   - Add JavaScript to generate a random 6-digit zero-padded code, e.g.:  
     ```js
     const code = Math.floor(100000 + Math.random() * 900000).toString();
     return [{ json: { ...items[0].json, code } }];
     ```  
   - Connect output from `Email Form` to this node.

4. **Add Google Sheets Node to Store Data:**  
   - Name: `Store Data`  
   - Type: Google Sheets  
   - Operation: Append or Update row  
   - Configure with your Google Sheet ID and Tab name.  
   - Map columns: Start Date (current timestamp), ID (workflow execution ID), Accepts Terms, Email, Code.  
   - Connect output from `Generate Code` to this node.  
   - Ensure Google Sheets OAuth2 credentials are set up and selected here.

5. **Add Email Send Node:**  
   - Name: `Send Email`  
   - Type: Email Send (SMTP)  
   - Configure SMTP credentials in n8n beforehand (host, port, username, password).  
   - From: `no-reply@yourdomain.com` or your domain email.  
   - To: Use email from previous node data.  
   - Subject: Customize, e.g., "Your Verification Code".  
   - Body: Include the generated `code` with instructions.  
   - Connect output from `Store Data` to this node.

6. **Create Verification Form Node:**  
   - Name: `Verification Form`  
   - Type: Form  
   - Field: Input for verification code.  
   - Connect output from `Send Email` to this node.

7. **Add If Node to Check Code:**  
   - Name: `Check Code`  
   - Type: If (Conditional)  
   - Condition: Compare user input code from `Verification Form` with stored code in Google Sheets (may require a lookup node before to fetch stored code).  
   - Connect output from `Verification Form` to this node.  
   - On True: Connect to `Main Form`.  
   - On False: Connect to `Incorrect Code Form`.

8. **Add Incorrect Code Form Node:**  
   - Name: `Incorrect Code Form`  
   - Type: Form  
   - Shows error message for invalid code and prompts retry.  
   - Connect false output of `Check Code` here.

9. **Add Second Check Node:**  
   - Name: `Second Check`  
   - Type: If  
   - Logic: Check if retry count exceeded (requires storing and checking attempt count, e.g., in Google Sheets or workflow variables).  
   - Connect output of `Incorrect Code Form` to this node.  
   - On True (retry allowed): Connect back to `Main Form` or `Verification Form` for retry.  
   - On False (too many retries): Connect to `Reset Form`.

10. **Add Reset Form Node:**  
    - Name: `Reset Form`  
    - Type: Form  
    - Allows user to restart the process.  
    - Connect false output of `Second Check` here.  
    - Connect output to `Email Form` to restart.

11. **Add Main Form Node:**  
    - Name: `Main Form`  
    - Type: Form  
    - The main application form accessible after verification.  
    - Connect true output of `Check Code` and true output of retry from `Second Check` here.

12. **Add NoOp Node:**  
    - Name: `Continue With Your Flow`  
    - Type: No Operation  
    - Connect output of `Main Form` here for further workflow customization.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Replace `YOUR_GOOGLE_SHEET_ID` with your actual Google Sheets ID in the "Store Data" node. | Google Sheets Setup |
| Configure SMTP credentials carefully to avoid email delivery failures. | SMTP Configuration |
| Test the entire flow first using a personal email before going live. | Best Practices |
| Consider implementing code expiration for enhanced security (advanced feature). | Security Features |
| Sticky notes within the workflow provide detailed explanations per node. | Workflow Comments |
| GDPR compliance ensured by explicit user consent and email verification. | Legal Considerations |
| Use this workflow for newsletters, registrations, lead generation, event sign-ups. | Use Cases |

---

This document fully describes the structure, logic, and operational flow of the "🔐 Double Opt-In Email Verification System with Google Sheets" workflow, enabling reproduction, modification, and troubleshooting by developers and AI agents.