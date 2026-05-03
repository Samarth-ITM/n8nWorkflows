Unique QRcode coupon assignment and validation for Lead Generation with SuiteCRM

https://n8nworkflows.xyz/workflows/unique-qrcode-coupon-assignment-and-validation-for-lead-generation-with-suitecrm-2899


# Unique QRcode coupon assignment and validation for Lead Generation with SuiteCRM

### 1. Workflow Overview

This workflow automates the assignment and validation of unique QR code coupons within a lead generation system integrated with SuiteCRM. It ensures that each coupon is uniquely assigned to a lead, prevents duplicate leads, validates coupon usage, and manages communication via email with QR code attachments.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and QR Code Extraction**  
  Receives incoming requests via webhook and extracts QR code data.

- **1.2 QR Code Validation and Lookup**  
  Validates the presence of a QR code, checks its existence in Google Sheets, and verifies if it has been used.

- **1.3 Lead Submission and Duplication Check**  
  Handles new lead form submissions, extracts form fields, and checks for duplicate leads based on email.

- **1.4 Coupon Assignment and Lead Creation**  
  Retrieves an unassigned coupon, authenticates with SuiteCRM, creates a new lead with the coupon, and updates Google Sheets.

- **1.5 QR Code Generation and Email Notification**  
  Generates a QR code image URL and sends an email to the lead with the QR code.

- **1.6 Response Handling**  
  Sends appropriate webhook responses based on coupon validation and usage status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and QR Code Extraction

- **Overview:**  
  This block listens for incoming HTTP requests containing QR code data and extracts the QR code value for further processing.

- **Nodes Involved:**  
  - Webhook  
  - Set coupon

- **Node Details:**

  - **Webhook**  
    - Type: Webhook Trigger  
    - Role: Entry point for incoming HTTP requests with QR code data as query parameters.  
    - Configuration: Listens on path `bb832325-8c58-4717-b866-41f8a9714cf2`, response mode set to use a response node.  
    - Inputs: External HTTP request  
    - Outputs: JSON data containing query parameters  
    - Edge Cases: Missing or malformed requests; webhook URL must be correctly configured in deployment.

  - **Set coupon**  
    - Type: Set  
    - Role: Extracts the `qr` parameter from the webhook query string and assigns it to a variable `qr`.  
    - Configuration: Assigns `qr` = `{{$json.query.qr}}`  
    - Inputs: Webhook output  
    - Outputs: JSON with `qr` field  
    - Edge Cases: If `qr` parameter is missing, subsequent validation will handle it.

---

#### 2.2 QR Code Validation and Lookup

- **Overview:**  
  Validates the presence of the QR code, checks if it exists in the Google Sheets coupon database, and verifies if the coupon has already been used.

- **Nodes Involved:**  
  - If  
  - Get Lead  
  - Not used?  
  - Token SuiteCRM 1  
  - Update Lead  
  - Update coupon used  
  - Coupon OK  
  - Coupon KO  
  - No coupon

- **Node Details:**

  - **If**  
    - Type: Conditional (If)  
    - Role: Checks if the `qr` field exists and is non-empty.  
    - Configuration: Condition tests if `qr` exists (`string operation: exists`)  
    - Inputs: Output from Set coupon  
    - Outputs:  
      - True: Proceed to Get Lead  
      - False: Respond with "Coupon not valid"  
    - Edge Cases: Empty or missing QR code leads to early termination.

  - **Get Lead**  
    - Type: Google Sheets (Lookup)  
    - Role: Searches Google Sheets for a row where `COUPON` equals the scanned QR code.  
    - Configuration: Document ID and sheet name set to the coupon database; returns first match.  
    - Inputs: If node (true branch)  
    - Outputs: Lead data associated with the coupon or empty if not found.  
    - Edge Cases: Google Sheets API errors, missing coupon entries.

  - **Not used?**  
    - Type: Conditional (If)  
    - Role: Checks if the coupon has not been used by verifying the "USED COUPON?" field is empty and row exists.  
    - Configuration:  
      - Condition 1: "USED COUPON?" field is empty  
      - Condition 2: Row number exists  
    - Inputs: Get Lead output  
    - Outputs:  
      - True: Proceed to Token SuiteCRM 1 (to update coupon usage)  
      - False: Respond with "Coupon already used"  
    - Edge Cases: Missing fields or inconsistent data in Google Sheets.

  - **Token SuiteCRM 1**  
    - Type: HTTP Request  
    - Role: Obtains OAuth2 access token from SuiteCRM for authenticated API calls.  
    - Configuration: POST to `/Api/access_token` with client credentials grant type, using environment variables for URL, client ID, and secret.  
    - Inputs: Not used? node (true branch)  
    - Outputs: Access token JSON  
    - Edge Cases: Authentication failures, network errors.

  - **Update Lead**  
    - Type: HTTP Request  
    - Role: Updates the lead in SuiteCRM to mark the coupon as used (`coupon_used_c` = "yes").  
    - Configuration: PATCH request to SuiteCRM API with lead ID from Get Lead node; authorization header uses token from Token SuiteCRM 1.  
    - Inputs: Token SuiteCRM 1 output  
    - Outputs: Updated lead data  
    - Edge Cases: API errors, invalid lead ID.

  - **Update coupon used**  
    - Type: Google Sheets (Update)  
    - Role: Updates the Google Sheets row for the lead to mark the coupon as used ("USED COUPON?" = "yes").  
    - Configuration: Matches row by "ID LEAD" and updates "USED COUPON?" field.  
    - Inputs: Update Lead output  
    - Outputs: Confirmation of update  
    - Edge Cases: Google Sheets API errors, row matching failures.

  - **Coupon OK**  
    - Type: Respond to Webhook  
    - Role: Sends a plain text response "Qr valid!" indicating successful coupon validation and usage update.  
    - Inputs: Update coupon used output  
    - Outputs: HTTP response  
    - Edge Cases: Response failures if webhook connection lost.

  - **Coupon KO**  
    - Type: Respond to Webhook  
    - Role: Sends "Coupon already used" response if coupon was found but marked as used.  
    - Inputs: Not used? node (false branch)  
    - Outputs: HTTP response

  - **No coupon**  
    - Type: Respond to Webhook  
    - Role: Sends "Coupon not valid" response if no QR code was provided or found.  
    - Inputs: If node (false branch)  
    - Outputs: HTTP response

---

#### 2.3 Lead Submission and Duplication Check

- **Overview:**  
  Handles new lead submissions from a form, extracts form data, and checks for duplicate leads by email in Google Sheets.

- **Nodes Involved:**  
  - On form submission  
  - Form Fields  
  - Duplicate Lead?  
  - Is Duplicate?  
  - Get Coupon

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point triggered when a user submits the lead generation form.  
    - Configuration: Form titled "Landing" with required fields: Name, Surname, Email (email type), Phone.  
    - Inputs: External form submission  
    - Outputs: JSON with form data  
    - Edge Cases: Missing required fields, malformed submissions.

  - **Form Fields**  
    - Type: Set  
    - Role: Extracts and assigns form fields into variables for easier access downstream.  
    - Configuration: Assigns Name, Surname, Email, Phone from form submission JSON.  
    - Inputs: On form submission output  
    - Outputs: Structured JSON with form fields  
    - Edge Cases: None significant.

  - **Duplicate Lead?**  
    - Type: Google Sheets (Lookup)  
    - Role: Checks Google Sheets for existing leads with the same email to prevent duplicates.  
    - Configuration: Looks up "EMAIL" column for the submitted email in the lead database sheet.  
    - Inputs: Form Fields output  
    - Outputs: Matching lead data or empty if none found  
    - Edge Cases: Google Sheets API errors, email format issues.

  - **Is Duplicate?**  
    - Type: Conditional (If)  
    - Role: Determines if a duplicate lead exists by checking if the email field from Duplicate Lead? is non-empty.  
    - Configuration: Condition tests if `EMAIL` field is not empty.  
    - Inputs: Duplicate Lead? output  
    - Outputs:  
      - True (duplicate exists): Ends flow (no further coupon assignment)  
      - False (no duplicate): Proceeds to Get Coupon  
    - Edge Cases: False negatives if email case sensitivity or formatting differs.

  - **Get Coupon**  
    - Type: Google Sheets (Lookup)  
    - Role: Retrieves the first available unassigned coupon (no lead assigned) from the coupon database.  
    - Configuration: Returns first match where "ID LEAD" is empty (filter implied).  
    - Inputs: Is Duplicate? (false branch)  
    - Outputs: Coupon data for assignment  
    - Edge Cases: No available coupons, Google Sheets API errors.

---

#### 2.4 Coupon Assignment and Lead Creation

- **Overview:**  
  Assigns the retrieved coupon to the new lead by authenticating with SuiteCRM, creating the lead record, and updating Google Sheets.

- **Nodes Involved:**  
  - Token SuiteCRM  
  - Create Lead SuiteCRM  
  - Update Sheet

- **Node Details:**

  - **Token SuiteCRM**  
    - Type: HTTP Request  
    - Role: Obtains OAuth2 access token from SuiteCRM for lead creation API calls.  
    - Configuration: POST to `/Api/access_token` with client credentials grant type, using environment variables for URL, client ID, and secret.  
    - Inputs: Get Coupon output  
    - Outputs: Access token JSON  
    - Edge Cases: Authentication failures, network errors.

  - **Create Lead SuiteCRM**  
    - Type: HTTP Request  
    - Role: Creates a new lead in SuiteCRM with form data and assigned coupon code.  
    - Configuration:  
      - POST to `/Api/V8/module` with JSON body containing lead attributes: first_name, last_name, email1, phone_mobile, coupon_c (coupon code).  
      - Authorization header uses token from Token SuiteCRM.  
    - Inputs: Token SuiteCRM output  
    - Outputs: Created lead data including lead ID  
    - Edge Cases: API errors, invalid data, token expiration.

  - **Update Sheet**  
    - Type: Google Sheets (Update)  
    - Role: Updates the coupon database sheet to assign the coupon to the newly created lead and record lead details.  
    - Configuration:  
      - Matches row by "COUPON" column.  
      - Updates columns: DATE (current timestamp), NAME, SURNAME, EMAIL, PHONE, COUPON, ID LEAD (from SuiteCRM response).  
    - Inputs: Create Lead SuiteCRM output  
    - Outputs: Confirmation of update  
    - Edge Cases: Google Sheets API errors, data mismatch.

---

#### 2.5 QR Code Generation and Email Notification

- **Overview:**  
  Generates a QR code image URL for the assigned coupon and sends an email to the lead with the QR code embedded.

- **Nodes Involved:**  
  - Get QR  
  - Send Email

- **Node Details:**

  - **Get QR**  
    - Type: HTTP Request  
    - Role: Generates a QR code image URL using an external QR code service (quickchart.io) embedding the coupon URL.  
    - Configuration:  
      - URL template: `https://quickchart.io/qr?text=https://n8n.n3witalia.com/webhook-test/bb832325-8c58-4717-b866-41f8a9714cf2?qr={{ coupon_code }}&size=250`  
      - Uses coupon code from Get Coupon node.  
    - Inputs: Update Sheet output  
    - Outputs: QR code image URL or binary image data (depending on service)  
    - Edge Cases: External service downtime, URL encoding issues.

  - **Send Email**  
    - Type: Email Send  
    - Role: Sends an HTML email to the lead with the QR code image embedded.  
    - Configuration:  
      - Subject: "[n8n] Exclusive Discount Coupon"  
      - To: Lead email from form fields  
      - From: Configured SMTP email address  
      - HTML body includes greeting, instructions, and embedded QR code image URL.  
    - Inputs: Get QR output  
    - Outputs: Email send confirmation  
    - Edge Cases: SMTP authentication errors, invalid email addresses, email delivery failures.

---

#### 2.6 Response Handling

- **Overview:**  
  Sends appropriate HTTP responses back to the webhook caller based on coupon validation and usage status.

- **Nodes Involved:**  
  - Coupon OK  
  - Coupon KO  
  - No coupon

- **Node Details:**

  - **Coupon OK**  
    - Sends "Qr valid!" text response when coupon is valid and marked as used.

  - **Coupon KO**  
    - Sends "Coupon already used" text response when coupon was found but already used.

  - **No coupon**  
    - Sends "Coupon not valid" text response when no QR code was provided or found.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                                    | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|-------------------------|---------------------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Webhook             | Webhook Trigger         | Receives incoming HTTP requests with QR code     | -                        | Set coupon              |                                                                                              |
| Set coupon          | Set                     | Extracts `qr` parameter from webhook query        | Webhook                  | If                      | Check if the QR code scan is valid                                                           |
| If                  | Conditional (If)        | Checks if QR code exists                           | Set coupon               | Get Lead, No coupon      | Check if coupon is valid                                                                     |
| Get Lead            | Google Sheets Lookup    | Finds lead by coupon code                          | If (true)                | Not used?                |                                                                                              |
| Not used?           | Conditional (If)        | Checks if coupon is unused                         | Get Lead                 | Token SuiteCRM 1, Coupon KO |                                                                                              |
| Token SuiteCRM 1    | HTTP Request            | Gets SuiteCRM token for updating lead             | Not used? (true)         | Update Lead              | Enter the lead with the relative coupon on Suite CRM. Change SUITECRMURL, CLIENTSECRET and CLIENTID |
| Update Lead         | HTTP Request            | Marks coupon as used in SuiteCRM                   | Token SuiteCRM 1         | Update coupon used       |                                                                                              |
| Update coupon used  | Google Sheets Update    | Marks coupon as used in Google Sheets              | Update Lead              | Coupon OK                |                                                                                              |
| Coupon OK           | Respond to Webhook      | Responds "Qr valid!"                               | Update coupon used       | -                       |                                                                                              |
| Coupon KO           | Respond to Webhook      | Responds "Coupon already used"                     | Not used? (false)        | -                       |                                                                                              |
| No coupon           | Respond to Webhook      | Responds "Coupon not valid"                        | If (false)               | -                       |                                                                                              |
| On form submission  | Form Trigger            | Receives new lead form submissions                 | -                        | Form Fields              |                                                                                              |
| Form Fields         | Set                     | Extracts form fields                               | On form submission       | Duplicate Lead?          | Check if the lead has already received the coupon                                            |
| Duplicate Lead?     | Google Sheets Lookup    | Checks for existing lead by email                  | Form Fields              | Is Duplicate?            |                                                                                              |
| Is Duplicate?       | Conditional (If)        | Determines if lead is duplicate                    | Duplicate Lead?           | (empty), Get Coupon      |                                                                                              |
| Get Coupon          | Google Sheets Lookup    | Retrieves first available unassigned coupon        | Is Duplicate? (false)    | Token SuiteCRM           | Find the first available unassigned coupon                                                   |
| Token SuiteCRM      | HTTP Request            | Gets SuiteCRM token for lead creation              | Get Coupon               | Create Lead SuiteCRM     | Enter the lead with the relative coupon on Suite CRM. Change SUITECRMURL, CLIENTSECRET and CLIENTID |
| Create Lead SuiteCRM| HTTP Request            | Creates lead in SuiteCRM with coupon                | Token SuiteCRM           | Update Sheet             |                                                                                              |
| Update Sheet        | Google Sheets Update    | Updates coupon assignment and lead info            | Create Lead SuiteCRM     | Get QR                   |                                                                                              |
| Get QR              | HTTP Request            | Generates QR code image URL                         | Update Sheet             | Send Email               | Set the Webhook URL                                                                          |
| Send Email          | Email Send              | Sends email with QR code to lead                    | Get QR                   | -                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `bb832325-8c58-4717-b866-41f8a9714cf2`  
   - Response Mode: Use response node  
   - Purpose: Receive incoming requests with QR code query parameter.

2. **Create Set Node "Set coupon"**  
   - Extract `qr` from webhook query parameters: `qr = {{$json.query.qr}}`  
   - Connect Webhook → Set coupon.

3. **Create If Node "If"**  
   - Condition: Check if `qr` exists and is not empty (`string operation: exists` on `{{$json.qr}}`)  
   - Connect Set coupon → If.

4. **Create Google Sheets Node "Get Lead"**  
   - Operation: Lookup row where `COUPON` equals `{{$json.qr}}`  
   - Document ID: Your Google Sheets coupon database ID  
   - Sheet Name: Corresponding sheet (e.g., `gid=0`)  
   - Return first match only  
   - Connect If (true) → Get Lead.

5. **Create If Node "Not used?"**  
   - Conditions:  
     - "USED COUPON?" field is empty (`string operation: empty`)  
     - Row number exists (`number operation: exists`)  
   - Connect Get Lead → Not used?.

6. **Create HTTP Request Node "Token SuiteCRM 1"**  
   - POST to `https://SUITECRMURL/Api/access_token`  
   - Body parameters:  
     - `grant_type`: `client_credentials`  
     - `client_id`: `CLIENTID`  
     - `client_secret`: `CLIENTSECRET`  
   - Connect Not used? (true) → Token SuiteCRM 1.

7. **Create HTTP Request Node "Update Lead"**  
   - PATCH to `https://SUITECRMURL/Api/V8/module`  
   - JSON body:  
     ```json
     {
       "data": {
         "type": "Leads",
         "id": "{{ $json['ID LEAD'] }}",
         "attributes": {
           "coupon_used_c": "yes"
         }
       }
     }
     ```  
   - Headers: Authorization Bearer token from Token SuiteCRM 1, Content-Type `application/vnd.api+json`  
   - Connect Token SuiteCRM 1 → Update Lead.

8. **Create Google Sheets Node "Update coupon used"**  
   - Operation: Update row matching "ID LEAD" with `"USED COUPON?" = "yes"`  
   - Document ID and Sheet Name same as Get Lead  
   - Connect Update Lead → Update coupon used.

9. **Create Respond to Webhook Node "Coupon OK"**  
   - Response Body: "Qr valid!"  
   - Connect Update coupon used → Coupon OK.

10. **Create Respond to Webhook Node "Coupon KO"**  
    - Response Body: "Coupon already used"  
    - Connect Not used? (false) → Coupon KO.

11. **Create Respond to Webhook Node "No coupon"**  
    - Response Body: "Coupon not valid"  
    - Connect If (false) → No coupon.

12. **Create Form Trigger Node "On form submission"**  
    - Form Title: "Landing"  
    - Fields: Name (text), Surname (text), Email (email), Phone (text), all required  
    - Purpose: Receive new lead submissions.

13. **Create Set Node "Form Fields"**  
    - Assign variables: Name, Surname, Email, Phone from form submission JSON  
    - Connect On form submission → Form Fields.

14. **Create Google Sheets Node "Duplicate Lead?"**  
    - Lookup row where "EMAIL" equals `{{$json.Email}}`  
    - Document ID and Sheet Name: Lead database sheet  
    - Connect Form Fields → Duplicate Lead?.

15. **Create If Node "Is Duplicate?"**  
    - Condition: Check if `EMAIL` field from Duplicate Lead? is not empty (`string operation: notEmpty`)  
    - Connect Duplicate Lead? → Is Duplicate?.

16. **Create Google Sheets Node "Get Coupon"**  
    - Lookup first row where "ID LEAD" is empty (unassigned coupon)  
    - Document ID and Sheet Name: Coupon database  
    - Connect Is Duplicate? (false) → Get Coupon.

17. **Create HTTP Request Node "Token SuiteCRM"**  
    - Same configuration as Token SuiteCRM 1  
    - Connect Get Coupon → Token SuiteCRM.

18. **Create HTTP Request Node "Create Lead SuiteCRM"**  
    - POST to `https://SUITECRMURL/Api/V8/module`  
    - JSON body with lead data and coupon:  
      ```json
      {
        "data": {
          "type": "Leads",
          "attributes": {
            "first_name": "{{ $json.Name }}",
            "last_name": "{{ $json.Surname }}",
            "email1": "{{ $json.Email }}",
            "phone_mobile": "{{ $json.Phone }}",
            "coupon_c": "{{ $json.COUPON }}"
          }
        }
      }
      ```  
    - Authorization header with token from Token SuiteCRM  
    - Connect Token SuiteCRM → Create Lead SuiteCRM.

19. **Create Google Sheets Node "Update Sheet"**  
    - Update row matching "COUPON" with lead data and current timestamp  
    - Fields: DATE (current datetime), NAME, SURNAME, EMAIL, PHONE, COUPON, ID LEAD (from SuiteCRM response)  
    - Document ID and Sheet Name: Coupon database  
    - Connect Create Lead SuiteCRM → Update Sheet.

20. **Create HTTP Request Node "Get QR"**  
    - GET request to QR code generator URL:  
      ```
      https://quickchart.io/qr?text=https://n8n.n3witalia.com/webhook-test/bb832325-8c58-4717-b866-41f8a9714cf2?qr={{ $json.COUPON }}&size=250
      ```  
    - Connect Update Sheet → Get QR.

21. **Create Email Send Node "Send Email"**  
    - SMTP credentials configured  
    - To: Lead email from form fields  
    - From: Your configured email address  
    - Subject: "[n8n] Exclusive Discount Coupon"  
    - HTML body includes greeting and embedded QR code image URL (same as Get QR URL)  
    - Connect Get QR → Send Email.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is a simple coupon assignment and validation system integrated with SuiteCRM and Google Sheets.                     | Workflow description                                                                                |
| DISCLAIMER: The system is basic and can be improved with additional controls and error handling.                                   | Sticky Note7 in workflow                                                                            |
| QR code images are generated using the free service QuickChart.io (https://quickchart.io/documentation/qr-codes/)                 | Get QR node configuration                                                                           |
| SuiteCRM API endpoints and credentials must be replaced with your actual SuiteCRM instance URL, client ID, and client secret.      | Token SuiteCRM and Create Lead SuiteCRM nodes                                                      |
| Google Sheets document ID and sheet names must be updated to match your coupon and lead databases.                                 | Google Sheets nodes configuration                                                                  |
| SMTP credentials must be configured to enable email sending.                                                                        | Send Email node                                                                                     |
| Webhook URL must be updated to match your n8n deployment environment for external access.                                           | Webhook node and QR code URL                                                                        |

---

This document provides a comprehensive understanding of the workflow structure, node configurations, and logical flow, enabling reproduction, modification, and troubleshooting by developers and automation agents.