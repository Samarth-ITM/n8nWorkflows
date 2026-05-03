Two-Factor Authentication with ClickSend Voice Calls and Email Verification

https://n8nworkflows.xyz/workflows/two-factor-authentication-with-clicksend-voice-calls-and-email-verification-8712


# Two-Factor Authentication with ClickSend Voice Calls and Email Verification

### 1. Workflow Overview

This workflow implements a Two-Factor Authentication system combining ClickSend voice calls with email verification. It is designed to verify a user’s phone number by sending a spoken numeric code through a voice call and verify the user's email address by sending a separate numeric code via email. The user must submit both codes correctly to complete verification.

**Target Use Cases:**  
- User identity verification for secure login or registration processes.  
- Scenarios requiring dual verification channels (phone and email) to increase security.

**Logical Blocks:**  
- **1.1 Form Input Reception:** Captures user input including phone number, voice preferences, email, and name.  
- **1.2 Voice Code Preparation and Delivery:** Creates a spoken verification code and sends it via ClickSend voice call API.  
- **1.3 Voice Code Verification:** Receives user input for the spoken code and validates it.  
- **1.4 Email Code Preparation and Delivery:** Sets an email verification code and sends it to the user’s email via SMTP.  
- **1.5 Email Code Verification:** Receives user input for the emailed code and validates it.  
- **1.6 Final Verification Outcome:** Determines success or failure based on code validation results and informs the user accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Input Reception

- **Overview:**  
Captures the user's input from a form including phone number, voice choice, language, email address, and name. This is the entry point for the workflow, triggered by a form submission.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **Node Name:** On form submission  
  - **Type and Role:** Form Trigger — listens for form submissions to start the workflow.  
  - **Configuration:**  
    - Form titled "Send Voice Message" with fields:  
      - "To" (phone number, required)  
      - "Voice" (dropdown: male/female, required)  
      - "Lang" (dropdown with multiple language options, required)  
      - "Email" (email field, required)  
      - "Nome" (name, required)  
    - Webhook ID provided for external triggering.  
  - **Inputs:** None (trigger)  
  - **Outputs:** Passes form data downstream.  
  - **Failure Modes:** Form submission errors, invalid or missing required fields.  
  - **Notes:** This node is the workflow’s external input interface.

#### 2.2 Voice Code Preparation and Delivery

- **Overview:**  
Prepares the voice verification code, formats it for clearer speech, and sends it via ClickSend voice call API using HTTP Basic Authentication.

- **Nodes Involved:**  
  - Set voice code  
  - Code for voice  
  - Send Voice  

- **Node Details:**  

  - **Set voice code**  
    - Type: Set node — assigns static voice verification code `"12345"`.  
    - Configuration: Sets field `Code` = "12345".  
    - Input: From On form submission  
    - Output: To Code for voice  
    - Notes: Code is hardcoded here for demonstration; should be dynamically generated in production.  

  - **Code for voice**  
    - Type: Code node — formats the `Code` by adding spaces between digits for better TTS clarity.  
    - Configuration: JavaScript loops over input, splits the code string into characters separated by spaces, and overwrites `Code`.  
    - Input: From Set voice code  
    - Output: To Send Voice  
    - Edge Cases: If `Code` field missing or empty, this node may output incorrect format.  

  - **Send Voice**  
    - Type: HTTP Request node — sends a POST request to ClickSend API to initiate the voice call.  
    - Configuration:  
      - URL: `https://rest.clicksend.com/v3/voice/send`  
      - Method: POST  
      - Body (JSON):  
        - `messages` array with one message object containing:  
          - `source`: "n8n"  
          - `body`: `"Your verification number is {{ $json.Code }}"` (uses spaced code)  
          - `to`: phone number from form submission  
          - `voice`: voice choice from form  
          - `lang`: language choice from form  
          - `machine_detection`: 1 (enables answering machine detection)  
      - Authentication: HTTP Basic Auth with credentials from ClickSend API key and username.  
      - Headers: Content-Type set to application/json  
    - Input: From Code for voice  
    - Output: To Verify voice code  
    - Edge Cases: API authentication failure, network timeout, invalid phone number format, API rate limits.  
    - Notes: Requires ClickSend account and API credentials configured in n8n.  

#### 2.3 Voice Code Verification

- **Overview:**  
Receives user input for the voice verification code and verifies it against the expected code.

- **Nodes Involved:**  
  - Verify voice code  
  - Is voice code correct?  
  - Fail voice code  

- **Node Details:**  

  - **Verify voice code**  
    - Type: Form node — presents a form for user to enter the code they heard on the call.  
    - Configuration: Single required field labeled "Verify".  
    - Input: From Send Voice  
    - Output: To Is voice code correct?  
    - Edge Cases: User submits empty or incorrect code.  

  - **Is voice code correct?**  
    - Type: If node — compares the user-entered code (`Verify`) with the expected code (`Code` set in Set voice code node).  
    - Configuration: Uses strict equality, case sensitive.  
    - Input: From Verify voice code  
    - Output:  
      - If TRUE: To Set email code (proceed to email verification)  
      - If FALSE: To Fail voice code (show failure message)  
    - Edge Cases: Mismatched data types or missing fields could cause false negatives.  

  - **Fail voice code**  
    - Type: Form node — displays a failure message when voice code verification fails.  
    - Configuration: Completion form with title "Oh no!" and message "Sorry, the code entered is invalid. Verification has not been completed".  
    - Input: From Is voice code correct? (FALSE branch)  
    - Output: None (end of this branch)  

#### 2.4 Email Code Preparation and Delivery

- **Overview:**  
Sets a separate static email verification code and sends it to the user’s email address using SMTP.

- **Nodes Involved:**  
  - Set email code  
  - Send Email  

- **Node Details:**  

  - **Set email code**  
    - Type: Set node — assigns static email verification code `"56789"`.  
    - Configuration: Sets field `Code Email` = "56789".  
    - Input: From Is voice code correct? (TRUE branch)  
    - Output: To Send Email  
    - Notes: Like voice code, this is hardcoded and should be generated dynamically for security.  

  - **Send Email**  
    - Type: Email Send node — sends an email with the verification code.  
    - Configuration:  
      - From Email: configured SMTP email address (configured in credentials)  
      - To Email: user email from form submission  
      - Subject: "Verify your code"  
      - Body (HTML): Personalized message including user name and the email code  
    - Input: From Set email code  
    - Output: To Verify email code  
    - Credentials: SMTP credentials configured in n8n (example: info@n3witalia.com)  
    - Edge Cases: SMTP authentication failure, email delivery issues, invalid email format.  

#### 2.5 Email Code Verification

- **Overview:**  
Receives user input for the email verification code and validates it against the expected code.

- **Nodes Involved:**  
  - Verify email code  
  - Is email code correct?  
  - Fail email code  

- **Node Details:**  

  - **Verify email code**  
    - Type: Form node — collects the email verification code from user input.  
    - Configuration: Single required field labeled "Verify email".  
    - Input: From Send Email  
    - Output: To Is email code correct?  
    - Edge Cases: Empty or invalid format input.  

  - **Is email code correct?**  
    - Type: If node — compares user input with stored code `Code Email`.  
    - Configuration: Strict equality check.  
    - Input: From Verify email code  
    - Output:  
      - TRUE: To Success node  
      - FALSE: To Fail email code node  
    - Edge Cases: Incorrect comparison due to data type or missing fields.  

  - **Fail email code**  
    - Type: Form node — shows failure message on email code verification failure.  
    - Configuration: Completion form with title "Oh no!" and message "Sorry, the code entered is invalid. Verification has not been completed".  
    - Input: From Is email code correct? (FALSE branch)  
    - Output: None (end of this branch)  

#### 2.6 Final Verification Outcome

- **Overview:**  
Displays a success message if both voice and email codes are correctly verified.

- **Nodes Involved:**  
  - Success  

- **Node Details:**  
  - Type: Form node — completion form that confirms successful verification.  
  - Configuration:  
    - Title: "Great!"  
    - Message: "Your mobile number and email address have been verified successfully. Thank you!"  
  - Input: From Is email code correct? (TRUE branch)  
  - Output: None (workflow ends here)  

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                            | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                  |
|---------------------|-------------------|--------------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger      | Captures user input from form               | -                        | Set voice code             |                                                                                              |
| Set voice code      | Set               | Sets static voice verification code "12345" | On form submission        | Code for voice             | Set the code that will be spoken in the verification phone call                              |
| Code for voice      | Code              | Adds spaces between digits for TTS clarity | Set voice code            | Send Voice                 |                                                                                              |
| Send Voice          | HTTP Request      | Calls ClickSend API to send voice call      | Code for voice            | Verify voice code          | Register at ClickSend and configure Basic Auth with API key (https://clicksend.com/?u=586989)|
| Verify voice code   | Form              | Receives user input for voice code          | Send Voice                | Is voice code correct?     |                                                                                              |
| Is voice code correct? | If              | Checks if voice code matches expected       | Verify voice code         | Set email code / Fail voice code |                                                                                              |
| Fail voice code     | Form              | Shows failure message if voice code invalid | Is voice code correct?    | -                          |                                                                                              |
| Set email code      | Set               | Sets static email verification code "56789" | Is voice code correct? (TRUE branch) | Send Email               | Set the code that will be sent in the verification email                                   |
| Send Email          | Email Send        | Sends email with verification code          | Set email code            | Verify email code          | In the node "Send Email" set the sender email address                                        |
| Verify email code   | Form              | Receives user input for email code           | Send Email                | Is email code correct?     |                                                                                              |
| Is email code correct? | If              | Checks if email code matches expected        | Verify email code         | Success / Fail email code  |                                                                                              |
| Fail email code     | Form              | Shows failure message if email code invalid  | Is email code correct?    | -                          |                                                                                              |
| Success             | Form              | Shows success message after complete verification | Is email code correct? (TRUE branch) | -                        |                                                                                              |
| Sticky Note         | Sticky Note       | Instruction to register at ClickSend          | -                        | -                          | Register here to ClickSend and obtain API key and credits; configure Basic Auth accordingly   |
| Sticky Note1        | Sticky Note       | Instruction on form submission outcome        | -                        | -                          | Submit the form to receive the voice call                                                   |
| Sticky Note2        | Sticky Note       | Workflow description                          | -                        | -                          | Explains the workflow’s purpose combining ClickSend voice calls and email verification       |
| Sticky Note3        | Sticky Note       | Instruction on voice code setting              | -                        | -                          | Set the code that will be spoken in the verification phone call                              |
| Sticky Note4        | Sticky Note       | Instruction on email code setting              | -                        | -                          | Set the code that will be sent in the verification email                                    |
| Sticky Note5        | Sticky Note       | Instruction on verification code setup and email sender | -                        | -                          | Set the verification code for voice and email; configure sender in Send Email node          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node:**
   - Type: Form Trigger  
   - Configure webhook with title "Send Voice Message"  
   - Add required fields:  
     - "To" (phone number)  
     - "Voice" (dropdown: male, female)  
     - "Lang" (dropdown with language options: en-us, it-it, en-au, en-gb, de-de, es-es, fr-fr, is-is, da-dk, nl-nl, pl-pl, pt-br, ru-ru)  
     - "Email" (email field)  
     - "Nome" (text)  
   - Save node.

2. **Create "Set voice code" node:**
   - Type: Set  
   - Add field "Code" with value "12345" (string)  
   - Connect output of "On form submission" to input of this node.

3. **Create "Code for voice" node:**
   - Type: Code  
   - Paste JavaScript code to space out digits of the `Code` field:  
     ```js
     for (const item of $input.all()) {
       const code = item.json.Code;
       const spacedCode = code.split('').join(' ');
       item.json.Code = spacedCode;
     }
     return $input.all();
     ```  
   - Connect output of "Set voice code" to input of this node.

4. **Create "Send Voice" node:**
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://rest.clicksend.com/v3/voice/send`  
   - Authentication: HTTP Basic Auth with credentials (setup credentials with your ClickSend username and API key)  
   - Headers: Content-Type = application/json  
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
   - Connect output of "Code for voice" to input of this node.

5. **Create "Verify voice code" node:**
   - Type: Form  
   - Add one required field labeled "Verify"  
   - Connect output of "Send Voice" to input of this node.

6. **Create "Is voice code correct?" node:**
   - Type: If  
   - Condition: Check if `$('Set voice code').item.json.Code` equals `$json.Verify` (strict equality)  
   - Connect output of "Verify voice code" to input of this node.

7. **Create "Fail voice code" node:**
   - Type: Form  
   - Configure as completion form with title "Oh no!" and message "Sorry, the code entered is invalid. Verification has not been completed"  
   - Connect FALSE output of "Is voice code correct?" to this node.

8. **Create "Set email code" node:**
   - Type: Set  
   - Add field "Code Email" with value "56789" (string)  
   - Connect TRUE output of "Is voice code correct?" to this node.

9. **Create "Send Email" node:**
   - Type: Email Send  
   - Configure SMTP credentials with your email server info  
   - Configure From Email (e.g., your verified sender email)  
   - To Email: `={{ $('On form submission').item.json.Email }}`  
   - Subject: "Verify your code"  
   - Body (HTML):  
     ```html
     Hi {{ $('On form submission').item.json['Nome '] }},<br>
     The email verification code is <b>{{ $json['Code Email'] }}</b>
     ```  
   - Connect output of "Set email code" to input of this node.

10. **Create "Verify email code" node:**
    - Type: Form  
    - Add one required field labeled "Verify email"  
    - Connect output of "Send Email" to input of this node.

11. **Create "Is email code correct?" node:**
    - Type: If  
    - Condition: Check if `$('Set email code').item.json['Code Email']` equals `$json['Verify email']` (strict equality)  
    - Connect output of "Verify email code" to input of this node.

12. **Create "Fail email code" node:**
    - Type: Form  
    - Configure as completion form with title "Oh no!" and message "Sorry, the code entered is invalid. Verification has not been completed"  
    - Connect FALSE output of "Is email code correct?" to this node.

13. **Create "Success" node:**
    - Type: Form  
    - Configure as completion form with title "Great!" and message "Your mobile number and email address have been verified successfully. Thank you!"  
    - Connect TRUE output of "Is email code correct?" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Register at ClickSend and obtain your API Key and initial credits (2 € free)                             | https://clicksend.com/?u=586989                                                                |
| Configure Basic Auth in the "Send Voice" node with ClickSend username and API Key as password            | See Sticky Note on ClickSend API setup                                                         |
| The verification codes are statically set for demonstration purposes. In production, generate random codes | Important security note                                                                         |
| The workflow requires SMTP credentials configured in n8n for sending emails                             | Example used: SMTP info@n3witalia.com                                                          |
| Voice verification uses machine detection enabled for better call handling                               | ClickSend API option `machine_detection: 1`                                                    |
| Language and voice parameters for TTS are selectable via dropdown on the form                            | Options include multiple locale codes (en-us, it-it, etc.)                                     |

---

**Disclaimer:**  
The provided text derives exclusively from an n8n automation workflow. All data processed is legal and public. No illegal or offensive content is present. All API usage respects service provider policies.