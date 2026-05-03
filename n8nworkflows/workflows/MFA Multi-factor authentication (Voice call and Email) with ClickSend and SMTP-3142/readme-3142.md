MFA Multi-factor authentication (Voice call and Email) with ClickSend and SMTP

https://n8nworkflows.xyz/workflows/mfa-multi-factor-authentication--voice-call-and-email--with-clicksend-and-smtp-3142


# MFA Multi-factor authentication (Voice call and Email) with ClickSend and SMTP

### 1. Workflow Overview

This workflow automates a multi-factor authentication (MFA) process combining **voice call verification** and **email verification**. It targets scenarios where user identity confirmation is critical, such as secure sign-ups or sensitive transactions, by verifying both phone number ownership and email address validity.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures user input via a form submission, including phone number, voice preferences, language, email, and name.
- **1.2 Voice Call Verification Setup**: Defines and formats a verification code, then sends it via a voice call using the ClickSend API.
- **1.3 Voice Code Verification**: Prompts the user to enter the code received by voice call and validates it.
- **1.4 Email Verification Setup**: Defines an email verification code and sends it via SMTP email.
- **1.5 Email Code Verification**: Prompts the user to enter the email code and validates it.
- **1.6 Final Notification**: Notifies the user of success or failure based on verification results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input through a form with fields for phone number, voice type, language, email, and name. This data drives subsequent verification steps.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point capturing user inputs.  
    - *Configuration:*  
      - Form titled "Send Voice Message" with required fields:  
        - To (phone number, e.g., +39xxxx)  
        - Voice (dropdown: male, female)  
        - Lang (dropdown with multiple language options, e.g., en-us, it-it, fr-fr)  
        - Email (email type)  
        - Name (text)  
    - *Input/Output:* No input; outputs form data as JSON for downstream nodes.  
    - *Edge Cases:*  
      - Missing or invalid phone number or email could cause failures downstream.  
      - Form validation ensures required fields are filled.  
    - *Version:* 2.2

---

#### 2.2 Voice Call Verification Setup

- **Overview:**  
  Sets a fixed verification code, formats it for clarity in speech, and sends it via a voice call using ClickSend API.

- **Nodes Involved:**  
  - Set voice code  
  - Code for voice  
  - Send Voice  
  - Sticky Notes (for instructions)

- **Node Details:**

  - **Set voice code**  
    - *Type:* Set  
    - *Role:* Defines the verification code for voice call ("12345").  
    - *Configuration:* Assigns string "12345" to field `Code`.  
    - *Input:* Receives form data.  
    - *Output:* Adds `Code` field to JSON.  
    - *Edge Cases:* Hardcoded code; consider dynamic generation for production.  
    - *Version:* 3.4

  - **Code for voice**  
    - *Type:* Code (JavaScript)  
    - *Role:* Formats the verification code by inserting spaces between characters for better TTS clarity.  
    - *Configuration:* Splits `Code` string into characters and rejoins with spaces.  
    - *Key Expression:* `code.split('').join(' ')`  
    - *Input:* Receives JSON with `Code`.  
    - *Output:* Modified JSON with spaced `Code`.  
    - *Edge Cases:* Assumes `Code` is a string; non-string input may cause errors.  
    - *Version:* 2

  - **Send Voice**  
    - *Type:* HTTP Request  
    - *Role:* Sends the voice call via ClickSend API.  
    - *Configuration:*  
      - POST to `https://rest.clicksend.com/v3/voice/send`  
      - Body JSON includes:  
        - `source`: "n8n"  
        - `body`: "Your verification number is {{ $json.Code }}" (uses spaced code)  
        - `to`: phone number from form input  
        - `voice`: voice type from form input  
        - `lang`: language from form input  
        - `machine_detection`: 1 (detects answering machines)  
      - Authentication: HTTP Basic Auth with ClickSend credentials (username and API key)  
      - Headers: Content-Type application/json  
    - *Input:* Receives formatted `Code` and form data.  
    - *Output:* Response from ClickSend API.  
    - *Edge Cases:*  
      - Authentication errors if credentials invalid.  
      - API rate limits or network timeouts.  
      - Invalid phone number format.  
    - *Version:* 4.2  
    - *Credentials:* ClickSend API HTTP Basic Auth

---

#### 2.3 Voice Code Verification

- **Overview:**  
  Prompts the user to enter the code they heard during the voice call and verifies correctness.

- **Nodes Involved:**  
  - Verify voice code  
  - Is voice code correct?  
  - Fail voice code (failure notification)

- **Node Details:**

  - **Verify voice code**  
    - *Type:* Form  
    - *Role:* Collects user input for voice verification code.  
    - *Configuration:* Single required field labeled "Verify".  
    - *Input:* Triggered after voice call sent.  
    - *Output:* User-entered code in JSON field `Verify`.  
    - *Edge Cases:* User may enter incorrect or malformed code.  
    - *Version:* 1

  - **Is voice code correct?**  
    - *Type:* If  
    - *Role:* Compares user input (`Verify`) with predefined voice code (`Code`).  
    - *Configuration:* Checks equality, case-sensitive, strict type validation.  
    - *Input:* User input from Verify voice code node.  
    - *Output:*  
      - True branch: proceeds to email verification.  
      - False branch: triggers failure notification.  
    - *Edge Cases:* Case sensitivity may cause false negatives if user input differs in case.  
    - *Version:* 2.2

  - **Fail voice code**  
    - *Type:* Form (completion)  
    - *Role:* Notifies user of failed voice code verification.  
    - *Configuration:*  
      - Title: "Oh no!"  
      - Message: "Sorry, the code entered is invalid. Verification has not been completed"  
    - *Input:* False branch from Is voice code correct?  
    - *Output:* Completion message to user.  
    - *Version:* 1

---

#### 2.4 Email Verification Setup

- **Overview:**  
  Defines an email verification code and sends it to the user’s email address via SMTP.

- **Nodes Involved:**  
  - Set email code  
  - Send Email  
  - Sticky Notes (instructions)

- **Node Details:**

  - **Set email code**  
    - *Type:* Set  
    - *Role:* Defines the email verification code ("56789").  
    - *Configuration:* Assigns string "56789" to field `Code Email`.  
    - *Input:* True branch from voice code verification.  
    - *Output:* Adds `Code Email` field to JSON.  
    - *Edge Cases:* Hardcoded code; consider dynamic generation for production.  
    - *Version:* 3.4

  - **Send Email**  
    - *Type:* Email Send  
    - *Role:* Sends verification email with the code.  
    - *Configuration:*  
      - To: email address from form input  
      - From: configured sender email (must be set in node)  
      - Subject: "Verify your code"  
      - HTML body: personalized greeting with recipient name and bolded verification code  
      - SMTP credentials configured in n8n (e.g., SMTP info@n3witalia.com)  
    - *Input:* Receives `Code Email` and form data.  
    - *Output:* Email sending result.  
    - *Edge Cases:*  
      - SMTP authentication failure.  
      - Invalid email address.  
      - Email delivery delays or spam filtering.  
    - *Version:* 2.1  
    - *Credentials:* SMTP

---

#### 2.5 Email Code Verification

- **Overview:**  
  Prompts the user to enter the email verification code and validates it.

- **Nodes Involved:**  
  - Verify email code  
  - Is email code correct?  
  - Success (final success notification)  
  - Fail email code (failure notification)

- **Node Details:**

  - **Verify email code**  
    - *Type:* Form  
    - *Role:* Collects user input for email verification code.  
    - *Configuration:* Single required field labeled "Verify email".  
    - *Input:* After email sent.  
    - *Output:* User-entered code in JSON field `Verify email`.  
    - *Edge Cases:* User may enter incorrect or malformed code.  
    - *Version:* 1

  - **Is email code correct?**  
    - *Type:* If  
    - *Role:* Compares user input (`Verify email`) with predefined email code (`Code Email`).  
    - *Configuration:* Checks equality, case-sensitive, strict type validation.  
    - *Input:* User input from Verify email code node.  
    - *Output:*  
      - True branch: triggers success notification.  
      - False branch: triggers failure notification.  
    - *Edge Cases:* Case sensitivity may cause false negatives.  
    - *Version:* 2.2

  - **Success**  
    - *Type:* Form (completion)  
    - *Role:* Notifies user of successful verification of both phone and email.  
    - *Configuration:*  
      - Title: "Great!"  
      - Message: "Your mobile number and email address have been verified successfully. Thank you!"  
    - *Input:* True branch from Is email code correct?  
    - *Output:* Completion message to user.  
    - *Version:* 1

  - **Fail email code**  
    - *Type:* Form (completion)  
    - *Role:* Notifies user of failed email code verification.  
    - *Configuration:*  
      - Title: "Oh no!"  
      - Message: "Sorry, the code entered is invalid. Verification has not been completed"  
    - *Input:* False branch from Is email code correct?  
    - *Output:* Completion message to user.  
    - *Version:* 1

---

#### 2.6 Sticky Notes (Documentation & Instructions)

- **Overview:**  
  Provides inline documentation and setup instructions for users configuring the workflow.

- **Nodes Involved:**  
  - Sticky Note (ClickSend registration and API key setup)  
  - Sticky Note1 (Form submission instructions)  
  - Sticky Note2 (Workflow purpose summary)  
  - Sticky Note3 (Voice code explanation)  
  - Sticky Note4 (Email code explanation)  
  - Sticky Note5 (Verification code setup and email sender instructions)

- **Node Details:**  
  - These nodes contain markdown-formatted text with links and setup steps.  
  - They do not affect workflow execution but are essential for user guidance.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                           | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|-------------------|-----------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger      | Captures user input from form           | -                      | Set voice code           |                                                                                              |
| Set voice code       | Set               | Defines voice verification code         | On form submission      | Code for voice           | Set the code that will be spoken in the verification phone call                              |
| Code for voice       | Code              | Formats code for TTS clarity             | Set voice code          | Send Voice               |                                                                                              |
| Send Voice           | HTTP Request      | Sends voice call via ClickSend API       | Code for voice          | Verify voice code        | Register at ClickSend and set Basic Auth with username and API key (see sticky note)         |
| Verify voice code    | Form              | Prompts user to enter voice code         | Send Voice              | Is voice code correct?   |                                                                                              |
| Is voice code correct?| If               | Validates entered voice code             | Verify voice code       | Set email code / Fail voice code |                                                                                              |
| Fail voice code      | Form (completion) | Notifies failure of voice code verification | Is voice code correct? (false) | -                       |                                                                                              |
| Set email code       | Set               | Defines email verification code          | Is voice code correct? (true) | Send Email               | Set the code that will be sent in the verification email                                    |
| Send Email           | Email Send        | Sends verification email via SMTP        | Set email code          | Verify email code        | In the node "Send Email" set the sender email and configure SMTP credentials                 |
| Verify email code    | Form              | Prompts user to enter email code          | Send Email              | Is email code correct?   |                                                                                              |
| Is email code correct?| If                | Validates entered email code              | Verify email code       | Success / Fail email code|                                                                                              |
| Success             | Form (completion) | Notifies successful verification          | Is email code correct? (true) | -                       |                                                                                              |
| Fail email code      | Form (completion) | Notifies failure of email code verification | Is email code correct? (false) | -                       |                                                                                              |
| Sticky Note          | Sticky Note       | Instructions for ClickSend API setup      | -                      | -                       | Register at ClickSend and obtain API key: https://clicksend.com/?u=586989                    |
| Sticky Note1         | Sticky Note       | Instructions for form submission          | -                      | -                       | Submit the form to receive a voice call with the verification code                          |
| Sticky Note2         | Sticky Note       | Workflow purpose summary                   | -                      | -                       | This workflow automates voice call and email verification using ClickSend and SMTP          |
| Sticky Note3         | Sticky Note       | Voice code explanation                     | -                      | -                       | Set the code that will be spoken in the verification phone call                             |
| Sticky Note4         | Sticky Note       | Email code explanation                     | -                      | -                       | Set the code that will be sent in the verification email                                   |
| Sticky Note5         | Sticky Note       | Verification code setup and email sender  | -                      | -                       | Set verification codes and sender email in respective nodes                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "On form submission":  
   - Form title: "Send Voice Message"  
   - Fields (all required):  
     - To (text, placeholder "+39xxxx")  
     - Voice (dropdown: male, female)  
     - Lang (dropdown: en-us, it-it, en-au, en-gb, de-de, es-es, fr-fr, is-is, da-dk, nl-nl, pl-pl, pt-br, ru-ru)  
     - Email (email type)  
     - Name (text, placeholder "Nome")

2. **Add a Set node** named "Set voice code":  
   - Add field `Code` (string) with value `"12345"` (hardcoded verification code).

3. **Add a Code node** named "Code for voice":  
   - JavaScript code to add spaces between characters of `Code`:  
     ```javascript
     for (const item of $input.all()) {
       const code = item.json.Code;
       const spacedCode = code.split('').join(' ');
       item.json.Code = spacedCode;
     }
     return $input.all();
     ```

4. **Add an HTTP Request node** named "Send Voice":  
   - Method: POST  
   - URL: `https://rest.clicksend.com/v3/voice/send`  
   - Authentication: HTTP Basic Auth (create credential with ClickSend username and API key)  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "messages": [
         {
           "source": "n8n",
           "body": "Your verification number is {{ $json.Code }}",
           "to": "{{ $('On form submission').item.json.To }}",
           "voice": "{{ $('On form submission').item.json.Voice }}",
           "lang": "{{ $('On form submission').item.json.Lang }}",
           "machine_detection": 1
         }
       ]
     }
     ```  
   - Send body and headers as JSON.

5. **Add a Form node** named "Verify voice code":  
   - Single required field labeled "Verify".

6. **Add an If node** named "Is voice code correct?":  
   - Condition: Check if `Verify` (user input) equals `Code` (from "Set voice code")  
   - Case sensitive, strict type validation.

7. **Add a Form (completion) node** named "Fail voice code":  
   - Title: "Oh no!"  
   - Message: "Sorry, the code entered is invalid. Verification has not been completed"  
   - Connect from false branch of "Is voice code correct?".

8. **Add a Set node** named "Set email code":  
   - Add field `Code Email` (string) with value `"56789"` (hardcoded email verification code)  
   - Connect from true branch of "Is voice code correct?".

9. **Add an Email Send node** named "Send Email":  
   - To: `{{ $('On form submission').item.json.Email }}`  
   - From: set your sender email address (e.g., "EMAIL")  
   - Subject: "Verify your code"  
   - HTML body:  
     ```html
     Hi {{ $('On form submission').item.json['Nome '] }},<br>
     The email verification code is <b>{{ $json['Code Email'] }}</b>
     ```  
   - Configure SMTP credentials in n8n and assign to this node.  
   - Connect from "Set email code".

10. **Add a Form node** named "Verify email code":  
    - Single required field labeled "Verify email".  
    - Connect from "Send Email".

11. **Add an If node** named "Is email code correct?":  
    - Condition: Check if `Verify email` (user input) equals `Code Email` (from "Set email code")  
    - Case sensitive, strict type validation.  
    - Connect from "Verify email code".

12. **Add a Form (completion) node** named "Success":  
    - Title: "Great!"  
    - Message: "Your mobile number and email address have been verified successfully. Thank you!"  
    - Connect from true branch of "Is email code correct?".

13. **Add a Form (completion) node** named "Fail email code":  
    - Title: "Oh no!"  
    - Message: "Sorry, the code entered is invalid. Verification has not been completed"  
    - Connect from false branch of "Is email code correct?".

14. **Connect nodes in the following order:**  
    - On form submission → Set voice code → Code for voice → Send Voice → Verify voice code → Is voice code correct?  
    - Is voice code correct? true → Set email code → Send Email → Verify email code → Is email code correct?  
    - Is voice code correct? false → Fail voice code  
    - Is email code correct? true → Success  
    - Is email code correct? false → Fail email code

15. **Add Sticky Notes (optional) for documentation:**  
    - Instructions for ClickSend API registration and credential setup.  
    - Explanation of verification codes.  
    - Form submission instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                         |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Register at ClickSend and obtain your API Key and free credits for testing.                      | https://clicksend.com/?u=586989                         |
| In the "Send Voice" node, create Basic Auth credentials with your ClickSend username and API Key.| Workflow setup instruction                              |
| SMTP credentials must be configured in n8n for sending verification emails.                      | SMTP setup in n8n                                      |
| The verification codes are hardcoded ("12345" for voice, "56789" for email) for demonstration.   | Consider dynamic code generation for production use.   |
| The workflow uses strict, case-sensitive code comparisons; user input must match exactly.        | Potential source of verification failures               |
| Voice call uses machine detection to avoid calls to answering machines.                           | ClickSend API feature                                   |
| Supported voice languages include en-us, it-it, fr-fr, and others as per form dropdown options.  | User selects preferred language for voice call          |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the MFA workflow combining voice call and email verification using ClickSend and SMTP in n8n.