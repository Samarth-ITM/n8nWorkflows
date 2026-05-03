N8N Contact Form Workflow

https://n8nworkflows.xyz/workflows/n8n-contact-form-workflow-4337


# N8N Contact Form Workflow

### 1. Workflow Overview

This workflow automates the processing of a website contact form submission using n8n. It targets scenarios where visitors submit inquiries or messages via a "Contact Us" form. The workflow validates inputs, sends notification emails to support, and manages user feedback via confirmation or error messages, including redirecting the user after submission.

Logical blocks included:

- **1.1 Input Reception & Validation:** Captures form submissions on a public webhook and handles input validation.
- **1.2 Email Notification:** Sends an email to support with contact details from the submission.
- **1.3 Email Delivery Confirmation:** Checks if the email was sent successfully and branches accordingly.
- **1.4 User Feedback Presentation:** Shows success or error confirmation forms to the submitter.
- **1.5 Redirection:** Redirects the user to a specified URL after successful form completion.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

- **Overview:** Listens for form submissions on a public webhook, collecting user input fields. If the submission fails or errors occur, the user is shown an error form.
- **Nodes Involved:**  
  - On form submission  
  - Form (Error Display)

- **Node Details:**

  - **On form submission**  
    - **Type:** Form Trigger  
    - **Role:** Entry point, exposes a webhook for public form submissions at path `/contact-us`.  
    - **Configuration:**  
      - Form titled "Contact Us" with fields: First Name, Last Name, Email (type email), Company, and Message (textarea).  
      - Required fields: First Name, Last Name, Email, Message.  
      - Ignores bot filtering is disabled (bots are not ignored).  
      - Uses workflow timezone.  
    - **Expressions:** Directly captures form field values into JSON.  
    - **Input/Output:** No inputs; outputs form submission data to next node.  
    - **Edge Cases:**  
      - Missing required fields cause form to reject submission by default.  
      - Webhook downtime or misconfiguration could cause data loss or submission failures.  
      - Bot submissions if `ignoreBots` is false may pollute data.  
    - **Version:** 2.2

  - **Form (Error Display)**  
    - **Type:** Form  
    - **Role:** Displays an error message if submission processing fails downstream.  
    - **Configuration:**  
      - Completion title: "Oops! Something Went Wrong"  
      - Completion message: "We couldn’t process your submission. Please try again later."  
    - **Input/Output:** Receives control from conditional node if email sending fails.  
    - **Edge Cases:** Only triggered on failure path. No additional validation.  
    - **Version:** 1

---

#### 1.2 Email Notification

- **Overview:** Sends an HTML email notification to support with details from the contact form submission.
- **Nodes Involved:**  
  - Send Email to Support

- **Node Details:**

  - **Send Email to Support**  
    - **Type:** Email Send  
    - **Role:** Constructs and sends a detailed email to notify support of the new contact inquiry.  
    - **Configuration:**  
      - SMTP credentials configured via "SMTP account".  
      - From email: notifications@your-domain.com  
      - To email: akhilgadiraju@email.com  
      - Subject includes submitter's first and last name dynamically: `🔔 New Contact Form Submission from {{ $json['First Name'] }} {{ $json['Last Name'] }}`  
      - HTML body uses a well-styled email template embedding all submitted fields, including first name, last name, email, company, and message, with appropriate formatting and design.  
    - **Expressions:** Utilizes `{{ $json['FieldName'] }}` syntax to inject form data into email content.  
    - **Input/Output:** Takes form submission JSON as input, outputs email sending result including `messageId`.  
    - **Edge Cases:**  
      - SMTP authentication failure or network issues could cause email sending to fail.  
      - Missing or malformed email addresses might cause bounce or rejection.  
      - Template errors (e.g., referencing wrong fields) could break email content.  
    - **Version:** 2.1

---

#### 1.3 Email Delivery Confirmation

- **Overview:** Evaluates if the email was successfully sent by checking the presence of a `messageId` in the email node output.  
- **Nodes Involved:**  
  - If Email Sent

- **Node Details:**

  - **If Email Sent**  
    - **Type:** If (Conditional)  
    - **Role:** Branches workflow based on whether the email sending was successful.  
    - **Configuration:**  
      - Checks if `messageId` exists and is non-empty in the previous node's output.  
      - If true, proceeds to success confirmation; if false, triggers error form output.  
    - **Expressions:** `={{ $json.messageId }}` with string existence operator.  
    - **Input/Output:** Input from "Send Email to Support"; outputs two branches: success and failure.  
    - **Edge Cases:**  
      - False negatives if email service returns no messageId despite sending email.  
      - Unexpected output structure from email node could cause expression failure.  
    - **Version:** 2.2

---

#### 1.4 User Feedback Presentation

- **Overview:** Presents a thank-you confirmation form on successful email delivery or an error form otherwise, then guides user to a redirect.  
- **Nodes Involved:**  
  - Confirmation Form  
  - Redirect Form  
  - End (Success)  
  - End (Error)

- **Node Details:**

  - **Confirmation Form**  
    - **Type:** Form  
    - **Role:** Shows a customized thank-you HTML message confirming receipt of the message.  
    - **Configuration:**  
      - Title: "✅ Thank You!"  
      - Button label: "Redirect to Home"  
      - Custom HTML content: Styled centered card with message "Your message has been received. We’ll get back to you shortly."  
    - **Input/Output:** Input from success branch of If node; outputs to Redirect Form.  
    - **Edge Cases:** Mostly static content; edge cases minimal.  
    - **Version:** 1

  - **Redirect Form**  
    - **Type:** Form  
    - **Role:** Redirects the user to a specified external URL after confirmation.  
    - **Configuration:**  
      - Operation: completion  
      - Redirect URL: https://www.linkedin.com/in/akhilv7/  
      - Respond with redirect status.  
    - **Input/Output:** Input from Confirmation Form; outputs to End (Success).  
    - **Edge Cases:**  
      - If external URL is down, user may see a broken redirect.  
      - Users with strict browser security settings may block redirects.  
    - **Version:** 1

  - **End (Success)**  
    - **Type:** NoOp (No Operation)  
    - **Role:** Terminates successful workflow execution.  
    - **Input/Output:** Input from Redirect Form; no outputs.  
    - **Version:** 1

  - **End (Error)**  
    - **Type:** NoOp  
    - **Role:** Terminates workflow on error path.  
    - **Input/Output:** Input from Error Form node; no outputs.  
    - **Version:** 1

---

### 3. Summary Table

| Node Name           | Node Type              | Functional Role                           | Input Node(s)          | Output Node(s)            | Sticky Note                                                                                      |
|---------------------|------------------------|-----------------------------------------|-----------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger           | Entry point; receives contact form data | None                  | Send Email to Support      | Customize Fields: Change the form fields, title, or descriptions in the formTrigger node.       |
| Send Email to Support| Email Send             | Sends notification email to support     | On form submission    | If Email Sent              | Customize Email: Update the email body or subject in the emailSend node.                        |
| If Email Sent        | If (Conditional)       | Checks if email was successfully sent   | Send Email to Support | Confirmation Form, Form    |                                                                                                 |
| Confirmation Form    | Form                   | Shows thank-you confirmation message    | If Email Sent (true)  | Redirect Form              | Customize Message Content: Change custom html content                                          |
| Redirect Form        | Form                   | Redirects user after submission          | Confirmation Form     | End (Success)              | Change Redirect URL: Redirect to a different URL by editing the Redirect Form node.             |
| End (Success)        | NoOp                   | Terminates successful workflow           | Redirect Form         | None                      |                                                                                                 |
| Form (Error Display) | Form                   | Shows error message on failure           | If Email Sent (false) | End (Error)                | Change Message: Configure the Error Message Output to Meet Your Implementation Requirements.    |
| End (Error)          | NoOp                   | Terminates workflow on error             | Form (Error Display)  | None                      |                                                                                                 |
| Sticky Note          | Sticky Note            | Notes and instructions                   | None                  | None                      |                                                                                                 |
| Sticky Note1         | Sticky Note            | Notes and instructions                   | None                  | None                      |                                                                                                 |
| Sticky Note2         | Sticky Note            | Notes and instructions                   | None                  | None                      |                                                                                                 |
| Sticky Note3         | Sticky Note            | Notes and instructions                   | None                  | None                      |                                                                                                 |
| Sticky Note4         | Sticky Note            | Notes and instructions                   | None                  | None                      |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node (Form Trigger):**  
   - Set operation to listen on webhook path `contact-us`.  
   - Title the form "Contact Us".  
   - Add form fields:  
     - First Name (required, placeholder "eg: John")  
     - Last Name (required, placeholder "eg: Doe")  
     - Email (type email, required, placeholder "eg: johndoe@gmail.com")  
     - Company (optional, placeholder "eg: Apple Inc")  
     - Message (textarea, required, placeholder "Type your message here ...")  
   - Disable bot ignoring.  
   - Use workflow timezone.

2. **Create "Send Email to Support" node (Email Send):**  
   - Connect input from "On form submission".  
   - Configure SMTP credentials (e.g., "SMTP account").  
   - From email: notifications@your-domain.com  
   - To email: akhilgadiraju@email.com  
   - Subject: `🔔 New Contact Form Submission from {{ $json['First Name'] }} {{ $json['Last Name'] }}`  
   - Paste the provided HTML email body template, embedding the form fields with expressions like `{{ $json['First Name'] }}`, `{{ $json['Message'] }}`, etc.

3. **Create "If Email Sent" node (If):**  
   - Connect input from "Send Email to Support".  
   - Set condition to check if `messageId` exists and is not empty: expression `={{ $json.messageId }}` with "exists" operator.

4. **Create "Confirmation Form" node (Form):**  
   - Connect input from If node's success branch.  
   - Set form title: "✅ Thank You!"  
   - Button label: "Redirect to Home"  
   - Insert HTML field with custom thank-you content:  
     ```
     <html>
     <head><meta charset="UTF-8"></head>
     <body style="margin:0;padding:0;background:linear-gradient(135deg,#e0f7ec,#f0fff5);height:100vh;display:flex;justify-content:center;align-items:center;">
       <div style="background:#fff;border:1px solid #d1e7dd;border-radius:12px;padding:50px 60px;box-shadow:0 6px 20px rgba(0,0,0,0.1);text-align:center;">
         <div style="font-size:20px;color:#333;">
           Your message has been received. We’ll get back to you shortly.
         </div>
       </div>
     </body>
     </html>
     ```

5. **Create "Redirect Form" node (Form):**  
   - Connect input from "Confirmation Form".  
   - Set operation to "completion".  
   - Set redirect URL to `https://www.linkedin.com/in/akhilv7/`.  
   - Set response type to "redirect".

6. **Create "End (Success)" node (NoOp):**  
   - Connect input from "Redirect Form".  
   - No configuration needed.

7. **Create "Form (Error Display)" node (Form):**  
   - Connect input from If node's failure branch.  
   - Set completion title: "Oops! Something Went Wrong"  
   - Set completion message: "We couldn’t process your submission. Please try again later."

8. **Create "End (Error)" node (NoOp):**  
   - Connect input from "Form (Error Display)".  
   - No additional configuration.

9. **Verify all connections:**

   - On form submission → Send Email to Support → If Email Sent  
   - If Email Sent (true) → Confirmation Form → Redirect Form → End (Success)  
   - If Email Sent (false) → Form (Error Display) → End (Error)

10. **Test the workflow:**

    - Deploy and test by submitting the form at `/contact-us`.  
    - Confirm email is received.  
    - Confirm the correct success or error feedback is shown.  
    - Confirm redirection works.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                           |
|----------------------------------------------------------------------------------------------------|------------------------------------------|
| Customize Fields: Change the form fields, title, or descriptions in the formTrigger node.          | Sticky Note near "On form submission"    |
| Customize Email: Update the email body or subject in the emailSend node.                           | Sticky Note near "Send Email to Support" |
| Customize Message Content: Change custom HTML content in the Confirmation Form node.               | Sticky Note near "Confirmation Form"     |
| Change Redirect URL: Redirect to a different URL by editing the Redirect Form node.                | Sticky Note near "Redirect Form"          |
| Change Message: Configure the Error Message Output to meet your implementation requirements.       | Sticky Note near Error Form node          |

---

**Disclaimer:** The provided text originates solely from an automated n8n workflow. It complies strictly with all applicable content policies and contains no illegal, offensive, or protected material. All handled data are legal and public.