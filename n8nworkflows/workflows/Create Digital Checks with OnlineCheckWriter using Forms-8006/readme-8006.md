Create Digital Checks with OnlineCheckWriter using Forms

https://n8nworkflows.xyz/workflows/create-digital-checks-with-onlinecheckwriter-using-forms-8006


# Create Digital Checks with OnlineCheckWriter using Forms

### 1. Workflow Overview

This workflow automates the creation of digital checks using the OnlineCheckWriter (OCW) API, driven by user input through forms. It is designed for businesses or individuals who want to issue checks programmatically by collecting necessary API credentials and check details interactively.

The workflow is logically divided into four blocks:

- **1.1 API Configuration:** Collects and stores the user’s OCW API credentials and bank account information.
- **1.2 Check Details Collection:** Gathers recipient and payment information needed for the check.
- **1.3 API Request Submission:** Sends the collected data to the OCW API to create the check.
- **1.4 Response Handling:** Returns confirmation and tracking details back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 API Configuration

- **Overview:**  
  This block captures the user’s OnlineCheckWriter API key, bank account ID, and a friendly account name. These credentials authenticate subsequent API requests.

- **Nodes Involved:**  
  - `OCW API Configuration` (Form Trigger)  
  - `Sticky Note` (Documentation)

- **Node Details:**

  - **OCW API Configuration**  
    - *Type & Role:* Form Trigger node that initiates the workflow upon form submission.  
    - *Configuration:*  
      - Path: unique webhook path for API config form.  
      - Form Title: "Setup OnlineCheckWriter Account."  
      - Fields:  
        - API Key (password, required)  
        - Bank Account ID (required)  
        - Account Name (required)  
      - Response mode: Responds immediately after submission.  
    - *Input/Output:* Trigger input via webhook, outputs form data JSON.  
    - *Edge Cases:*  
      - Missing required fields will block submission.  
      - Invalid or expired API keys not validated here but will cause failures downstream.  
      - Network issues during webhook receipt.  
    - *Sticky Note:* Explains the purpose of the form and required credentials.

  - **Sticky Note** (API Configuration)  
    - Provides detailed instructions on what to enter and why.

---

#### 2.2 Check Details Collection

- **Overview:**  
  Collects detailed payment information from the user such as payee name, address, contact info, payment amount, memo, and issue date. Validates required fields.

- **Nodes Involved:**  
  - `Check Details Form` (Form)  
  - `Sticky Note 2` (Documentation)

- **Node Details:**

  - **Check Details Form**  
    - *Type & Role:* Form node that collects check recipient and payment details.  
    - *Configuration:*  
      - Fields:  
        - Payee Name (required)  
        - Company Name (optional)  
        - Address Line 1 (required)  
        - Address Line 2 (optional)  
        - City (required)  
        - State (required)  
        - ZIP Code (required)  
        - Phone (required)  
        - Email (required)  
        - Amount (required)  
        - Memo (optional)  
        - Internal Note (optional)  
        - Reference ID (optional)  
        - Issue Date (required, date field)  
    - *Input/Output:* Triggered after API Configuration node; outputs JSON with all form fields.  
    - *Edge Cases:*  
      - Validation of required fields prevents submission.  
      - Email or phone formatting is not enforced here but may cause API errors.  
      - Incorrect date formatting may cause issues downstream.  
    - *Sticky Note 2:* Describes the intent and validation of this form.

---

#### 2.3 API Request Submission

- **Overview:**  
  Formats and sends a POST request to the OnlineCheckWriter API to create the check using the collected data and credentials.

- **Nodes Involved:**  
  - `Send Check via OCW API` (HTTP Request)  
  - `Sticky Note 3` (Documentation)

- **Node Details:**

  - **Send Check via OCW API**  
    - *Type & Role:* HTTP Request node to submit the check creation request to OCW API.  
    - *Configuration:*  
      - URL: `https://test.onlinecheckwriter.com/api/v3/quickpay/check` (test endpoint, switch to production when ready)  
      - Method: POST  
      - Body: JSON payload constructed from form inputs and stored credentials, including:  
        - Source account info (Bank Account ID)  
        - Destination payee details (name, address, phone, email)  
        - Payment details (amount, memo, note, reference ID, issue date)  
      - Headers:  
        - Authorization: Bearer token using API Key from configuration  
        - Content-Type: application/json  
        - Accept: application/json  
      - Timeout: 30 seconds  
      - Continue On Fail: true (to handle errors downstream)  
    - *Input:* JSON data from Check Details Form; credentials from API Configuration node.  
    - *Output:* JSON response from OCW API (success or error).  
    - *Edge Cases:*  
      - Authentication failure due to invalid API key.  
      - Timeout if API is slow or unavailable.  
      - Validation errors if required payment fields are missing or malformed.  
      - Network connectivity issues.  
    - *Sticky Note 3:* Explains API request configuration details and error tolerance.

---

#### 2.4 Response Handling

- **Overview:**  
  Sends a confirmation message back to the user with check details and tracking information after successful API submission.

- **Nodes Involved:**  
  - `Success Response1` (Respond to Webhook)  
  - `Sticky Note 4` (Documentation)

- **Node Details:**

  - **Success Response1**  
    - *Type & Role:* Node that responds to the original webhook call with a confirmation message.  
    - *Configuration:*  
      - HTTP response code: 200 (OK)  
      - Response type: Text  
      - Response body: Template message including:  
        - Check ID  
        - Amount  
        - Payee  
        - Status  
        - Link to track check on OCW platform (static URL mentioned in sticky note, but not dynamically inserted)  
    - *Input:* JSON response from API request node.  
    - *Output:* HTTP response to user.  
    - *Edge Cases:*  
      - If the API request failed, the response may contain error or incomplete data.  
      - No explicit error handling message; "continueOnFail" allows workflow to reach this node regardless.  
    - *Sticky Note 4:* Details what information is returned to the user.

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role               | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                               |
|----------------------|--------------------|------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note        | Documentation for API Config | —                      | —                       | ## 1. API Configuration: Explains API key, bank account ID, and account name needed for authentication.  |
| OCW API Configuration| Form Trigger       | Collect OCW API credentials  | —                      | Check Details Form       | ## 1. API Configuration: Form for entering API key, bank account ID, and account name.                   |
| Sticky Note 2        | Sticky Note        | Documentation for Check Form | —                      | —                       | ## 2. Check Details: Explains fields for payee and payment information and validation.                   |
| Check Details Form    | Form               | Collect check recipient info | OCW API Configuration  | Send Check via OCW API   | ## 2. Check Details: Form fields for payee data, amount, memo, issue date, etc.                           |
| Sticky Note 3        | Sticky Note        | Documentation for API Request| —                      | —                       | ## 3. API Request: Details on POST request to OCW API, including authentication and timeout.             |
| Send Check via OCW API| HTTP Request       | Submit check creation request| Check Details Form      | Success Response1        | ## 3. API Request: POST to OCW API test endpoint with Bearer token and JSON body.                        |
| Sticky Note 4        | Sticky Note        | Documentation for Response   | —                      | —                       | ## 4. Response: Describes confirmation message with check ID, amount, payee, status, and tracking link.  |
| Success Response1    | Respond to Webhook | Send confirmation back to user| Send Check via OCW API | —                       | ## 4. Response: Returns success message and details to the user after check creation.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node: "OCW API Configuration"**  
   - Type: Form Trigger  
   - Set webhook path (e.g., a unique UUID or descriptive string).  
   - Form Title: "Setup OnlineCheckWriter Account"  
   - Form Description: "One-time setup: Enter your API key and bank account details"  
   - Add form fields:  
     - API Key (Password, required)  
     - Bank Account ID (Required)  
     - Account Name (Required)  
   - Response Mode: Respond immediately upon submission.

2. **Create Form node: "Check Details Form"**  
   - Type: Form  
   - Add form fields:  
     - Payee Name (Required)  
     - Company Name (Optional)  
     - Address Line 1 (Required)  
     - Address Line 2 (Optional)  
     - City (Required)  
     - State (Required)  
     - ZIP Code (Required)  
     - Phone (Required)  
     - Email (Required)  
     - Amount (Required)  
     - Memo (Optional)  
     - Internal Note (Optional)  
     - Reference ID (Optional)  
     - Issue Date (Date field, Required)  
   - No special response mode needed.

3. **Connect "OCW API Configuration" node output to "Check Details Form" node input.**

4. **Create HTTP Request node: "Send Check via OCW API"**  
   - Type: HTTP Request  
   - URL: `https://test.onlinecheckwriter.com/api/v3/quickpay/check` (replace with production URL when ready)  
   - Method: POST  
   - Body Content Type: Raw JSON  
   - Construct body JSON using expressions referencing:  
     - Bank Account ID from `OCW API Configuration` form data  
     - Payee and payment fields from `Check Details Form` form data  
     Example body template:  
     ```json
     {
       "source": {
         "accountType": "bankaccount",
         "accountId": "{{ $('OCW API Configuration').item.json['Bank Account ID'] }}"
       },
       "destination": {
         "name": "{{ $json['Payee Name'] }}",
         "company": "{{ $json['Company Name'] }}",
         "address1": "{{ $json['Address Line 1'] }}",
         "address2": "{{ $json['Address Line 2'] }}",
         "city": "{{ $json.City }}",
         "state": "{{ $json.State }}",
         "zip": "{{ $json['ZIP Code'] }}",
         "phone": "{{ $json.Phone }}",
         "email": "{{ $json.Email }}"
       },
       "payment_details": {
         "amount": "{{ $json.Amount }}",
         "memo": "{{ $json.Memo }}",
         "note": "{{ $json['Internal Note'] }}",
         "referenceID": "{{ $json['Reference ID'] }}",
         "issueDate": "{{ $json['Issue Date'] }}"
       }
     }
     ```  
   - Headers:  
     - Authorization: Bearer token with API Key from `OCW API Configuration`  
     - Content-Type: application/json  
     - Accept: application/json  
   - Timeout: 30000 ms (30 seconds)  
   - Enable "Continue On Fail" to allow error handling downstream.

5. **Connect "Check Details Form" node output to "Send Check via OCW API" input.**

6. **Create Respond to Webhook node: "Success Response1"**  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Type: Text  
   - Response Body with template:  
     ```
     Check successfully created!

     Check ID: {{ $json.checkId }}
     Amount: ${{ $json.amount }}
     Payee: {{ $json.payee }}
     Status: {{ $json.status }}

     You can track your check at OnlineCheckWriter.com
     ```
   - This node will send confirmation back to the user.

7. **Connect "Send Check via OCW API" output to "Success Response1" input.**

8. **Add Sticky Note nodes as documentation at appropriate positions:**  
   - One explaining API configuration form and credentials.  
   - One explaining check details form fields.  
   - One explaining the API request configuration and error tolerance.  
   - One explaining the response message and tracking info.

9. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                               |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Use the OCW test endpoint for development; switch to production URL when ready.                                              | API URL: https://test.onlinecheckwriter.com/api/v3/quickpay/check |
| The workflow assumes valid API credentials and bank account IDs are used; invalid credentials will cause API request failure. | Authentication context                          |
| Timeout is set to 30 seconds to balance responsiveness and potential network delay.                                           | HTTP Request timeout setting                    |
| The workflow continues on API failure to allow graceful error handling or user feedback extensions.                          | HTTP Request node "Continue On Fail" parameter |
| Check tracking link is static in response text; consider dynamic linking using returned checkId for enhanced UX.             | Response message customization                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are lawful and public.