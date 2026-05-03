Drive-To-Store Lead Generation (with coupon) with Google Sheets & SuiteCRM

https://n8nworkflows.xyz/workflows/drive-to-store-lead-generation--with-coupon--with-google-sheets---suitecrm-2890


# Drive-To-Store Lead Generation (with coupon) with Google Sheets & SuiteCRM

### 1. Workflow Overview

This workflow automates a **Drive-To-Store Lead Generation** process that integrates a web form, Google Sheets, and SuiteCRM. Its primary purpose is to capture leads via a landing page form, assign unique coupons to new leads, prevent duplicate entries, and synchronize lead data with SuiteCRM and Google Sheets for tracking and marketing follow-up.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures form submissions from a landing page.
- **1.2 Form Data Extraction:** Extracts and prepares form data for processing.
- **1.3 Duplicate Lead Verification:** Checks if the lead email already exists in Google Sheets to avoid duplicates.
- **1.4 Coupon Retrieval:** Retrieves an available coupon code from Google Sheets for new leads.
- **1.5 SuiteCRM Lead Creation:** Authenticates with SuiteCRM and creates a new lead with the coupon code.
- **1.6 Google Sheets Update:** Updates the Google Sheets document with the new lead’s details and coupon assignment.
- **1.7 Webhook Response:** Sends success or failure responses back to the form submission webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits the landing page form, capturing the raw input data.

- **Nodes Involved:**  
  - On form submission  
  - Webhook

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Listens for form submissions on a landing page titled "Landing" with fields: Name, Surname, Email (email type), Phone.  
    - Configuration: All fields are required.  
    - Input: HTTP POST from the landing page form.  
    - Output: Raw form data JSON with user inputs.  
    - Edge Cases: Missing required fields will prevent submission; malformed email inputs may cause validation errors on the form side.

  - **Webhook**  
    - Type: Webhook  
    - Role: Receives the HTTP POST request from the form submission trigger (acts as an entry point).  
    - Configuration: HTTP POST method, unique webhook path.  
    - Input: HTTP request from the form.  
    - Output: Passes data to the next node.  
    - Edge Cases: Network issues or incorrect webhook URL configuration may cause failures.

#### 2.2 Form Data Extraction

- **Overview:**  
  Extracts and maps the form fields into named variables for easier reference downstream.

- **Nodes Involved:**  
  - Form Fields (Set node)

- **Node Details:**

  - **Form Fields**  
    - Type: Set  
    - Role: Maps raw form JSON fields into named variables: Name, Surname, Email, Phone.  
    - Configuration: Assigns string values from incoming JSON fields.  
    - Input: Raw form submission JSON.  
    - Output: JSON with explicit keys for each form field.  
    - Edge Cases: If input JSON keys are missing or renamed, expressions will fail.

#### 2.3 Duplicate Lead Verification

- **Overview:**  
  Checks if the submitted email already exists in the Google Sheets document to prevent duplicate leads.

- **Nodes Involved:**  
  - Duplicate Lead? (Google Sheets)  
  - Is Duplicate? (If node)  
  - Respond KO (Respond to Webhook)

- **Node Details:**

  - **Duplicate Lead?**  
    - Type: Google Sheets  
    - Role: Searches the Google Sheet for the submitted email in the "EMAIL" column.  
    - Configuration: Uses filter to lookup the email value from the form data.  
    - Input: Form Fields output JSON with Email.  
    - Output: Rows matching the email or empty if none found.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Google API quota limits, invalid credentials, or sheet access issues may cause failures.

  - **Is Duplicate?**  
    - Type: If  
    - Role: Checks if the Google Sheets query returned a non-empty result (indicating a duplicate).  
    - Configuration: Condition tests if the EMAIL field from Google Sheets is not empty.  
    - Input: Output from Duplicate Lead? node.  
    - Output:  
      - True branch: Duplicate found → Respond KO node.  
      - False branch: No duplicate → proceeds to coupon retrieval.  
    - Edge Cases: Expression errors if expected fields are missing.

  - **Respond KO**  
    - Type: Respond to Webhook  
    - Role: Sends a JSON response indicating a duplicate lead was detected.  
    - Configuration: HTTP 200 with JSON body `{ "result": "KO", "reason": "duplicate lead" }`.  
    - Input: Triggered only if duplicate detected.  
    - Output: Ends workflow for duplicates.  
    - Edge Cases: Network issues sending response.

#### 2.4 Coupon Retrieval

- **Overview:**  
  Retrieves an available coupon code from Google Sheets to assign to the new lead.

- **Nodes Involved:**  
  - Get Coupon (Google Sheets)  
  - Token SuiteCRM (HTTP Request)

- **Node Details:**

  - **Get Coupon**  
    - Type: Google Sheets  
    - Role: Retrieves the first available coupon code from the sheet.  
    - Configuration: Returns the first match; filters by "ID LEAD" column (empty or unassigned coupons).  
    - Input: Triggered if no duplicate found.  
    - Output: JSON containing the coupon code under key "COUPON".  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: No coupons available, API errors, or sheet access issues.

  - **Token SuiteCRM**  
    - Type: HTTP Request  
    - Role: Obtains OAuth2 access token from SuiteCRM using client credentials grant.  
    - Configuration: POST request to SuiteCRM `/Api/access_token` endpoint with body parameters: grant_type=client_credentials, client_id, client_secret.  
    - Input: Triggered after coupon retrieval.  
    - Output: JSON containing access_token.  
    - Edge Cases: Invalid credentials, network errors, SuiteCRM API downtime.

#### 2.5 SuiteCRM Lead Creation

- **Overview:**  
  Creates a new lead in SuiteCRM with the form data and assigned coupon code.

- **Nodes Involved:**  
  - Create Lead SuiteCRM (HTTP Request)

- **Node Details:**

  - **Create Lead SuiteCRM**  
    - Type: HTTP Request  
    - Role: Sends POST request to SuiteCRM API to create a lead record.  
    - Configuration:  
      - URL: SuiteCRM `/Api/V8/module` endpoint.  
      - Method: POST  
      - Headers: Authorization Bearer token from Token SuiteCRM node, Content-Type application/vnd.api+json  
      - Body: JSON with lead attributes: first_name, last_name, email1, phone_mobile, coupon_c (custom field).  
    - Input: Form Fields data and coupon code from Get Coupon node; token from Token SuiteCRM.  
    - Output: JSON response with created lead data including lead ID.  
    - Edge Cases: API errors, invalid token, missing required fields, network issues.

#### 2.6 Google Sheets Update

- **Overview:**  
  Updates the Google Sheets document with the newly created lead’s details, including coupon assignment and timestamp.

- **Nodes Involved:**  
  - Update Lead (Google Sheets)

- **Node Details:**

  - **Update Lead**  
    - Type: Google Sheets  
    - Role: Updates the row corresponding to the assigned coupon with lead details: Name, Surname, Email, Phone, Coupon, Lead ID, and current date/time.  
    - Configuration:  
      - Operation: Update  
      - Matching column: COUPON  
      - Columns mapped from SuiteCRM response and Get Coupon node.  
      - Date formatted as `dd/MM/yyyy HH:mm:ss`.  
    - Input: Output from Create Lead SuiteCRM node and Get Coupon node.  
    - Credentials: Google Sheets OAuth2.  
    - Edge Cases: Sheet access issues, concurrency conflicts, API limits.

#### 2.7 Webhook Response

- **Overview:**  
  Sends a success response back to the webhook indicating the lead was created successfully.

- **Nodes Involved:**  
  - Respond OK (Respond to Webhook)

- **Node Details:**

  - **Respond OK**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 with JSON `{ "result": "OK", "reason": "lead created" }`.  
    - Input: Triggered after successful lead creation and sheet update.  
    - Output: Ends workflow with success response.  
    - Edge Cases: Network issues sending response.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                          |
|---------------------|-------------------------|---------------------------------------|-------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Webhook             | Webhook                 | Entry point for form submission       | —                       | On form submission       |                                                                                                    |
| On form submission   | Form Trigger            | Captures form data                    | Webhook                 | Form Fields              |                                                                                                    |
| Form Fields         | Set                     | Extracts and maps form fields          | On form submission       | Duplicate Lead?          |                                                                                                    |
| Duplicate Lead?      | Google Sheets           | Checks for duplicate email in sheet   | Form Fields              | Is Duplicate?            | Check if the lead has already received the coupon                                                  |
| Is Duplicate?        | If                      | Branches workflow on duplicate check  | Duplicate Lead?          | Respond KO (true), Get Coupon (false) |                                                                                                    |
| Respond KO           | Respond to Webhook      | Sends duplicate lead response          | Is Duplicate? (true)     | —                        |                                                                                                    |
| Get Coupon           | Google Sheets           | Retrieves an available coupon          | Is Duplicate? (false)    | Token SuiteCRM           | Find the first available unassigned coupon                                                        |
| Token SuiteCRM       | HTTP Request            | Obtains SuiteCRM OAuth2 token          | Get Coupon               | Create Lead SuiteCRM     | Enter the lead with the relative coupon on Suite CRM. Change SUITECRMURL, CLIENTSECRET and CLIENTID. For full tutorial: https://docs.suitecrm.com/developer/api/developer-setup-guide/json-api/#_generate_private_and_public_key_for_oauth2 |
| Create Lead SuiteCRM | HTTP Request            | Creates lead in SuiteCRM                | Token SuiteCRM           | Update Lead              |                                                                                                    |
| Update Lead          | Google Sheets           | Updates Google Sheet with lead details | Create Lead SuiteCRM     | Respond OK               |                                                                                                    |
| Respond OK           | Respond to Webhook      | Sends success response                  | Update Lead              | —                        |                                                                                                    |
| Sticky Note          | Sticky Note             | Instructional note                      | —                       | —                        | ## STEP 1: Create a Google Sheet like this (Fill only the column "COUPON") See linked image and sheet. |
| Sticky Note1         | Sticky Note             | Instructional note                      | —                       | —                        | ## STEP 2 - MAIN FLOW: Workflow ideal for SuiteCRM integration and Google Sheets tracking. Use external form with webhook triggers. |
| Sticky Note2         | Sticky Note             | Instructional note                      | —                       | —                        | Check if the lead has already received the coupon                                                  |
| Sticky Note3         | Sticky Note             | Instructional note                      | —                       | —                        | Find the first available unassigned coupon                                                        |
| Sticky Note4         | Sticky Note             | Instructional note                      | —                       | —                        | Enter the lead with the relative coupon on Suite CRM. Change SUITECRMURL, CLIENTSECRET and CLIENTID. For full tutorial: https://docs.suitecrm.com/developer/api/developer-setup-guide/json-api/#_generate_private_and_public_key_for_oauth2 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "4b98315d-782e-47a5-8fea-7d16155c811d")  
   - Purpose: Entry point for form submissions.

2. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Connect input from Webhook node.  
   - Configure form title "Landing" with fields:  
     - Name (text, required)  
     - Surname (text, required)  
     - Email (email type, required)  
     - Phone (text, required)

3. **Create Set Node (Form Fields)**  
   - Connect input from Form Trigger node.  
   - Assign variables:  
     - Name = `{{$json.Name}}`  
     - Surname = `{{$json.Surname}}`  
     - Email = `{{$json.Email}}`  
     - Phone = `{{$json.Phone}}`

4. **Create Google Sheets Node (Duplicate Lead?)**  
   - Connect input from Set node.  
   - Operation: Lookup rows  
   - Document ID: Google Sheets document containing leads and coupons.  
   - Sheet Name: Sheet with leads data (e.g., "Foglio1").  
   - Filter: Lookup column "EMAIL" equals `={{$json.Email}}`  
   - Credentials: Configure Google Sheets OAuth2 credentials.

5. **Create If Node (Is Duplicate?)**  
   - Connect input from Duplicate Lead? node.  
   - Condition: Check if `{{$json.EMAIL}}` is not empty (indicating duplicate).  
   - True branch: Connect to Respond KO node.  
   - False branch: Connect to Get Coupon node.

6. **Create Respond to Webhook Node (Respond KO)**  
   - Connect input from Is Duplicate? node (true branch).  
   - Response Code: 200  
   - Response Body: `{ "result": "KO", "reason": "duplicate lead" }`

7. **Create Google Sheets Node (Get Coupon)**  
   - Connect input from Is Duplicate? node (false branch).  
   - Operation: Lookup rows, return first match.  
   - Filter: Lookup column "ID LEAD" (empty or unassigned).  
   - Document ID and Sheet Name: Same as Duplicate Lead? node.  
   - Credentials: Google Sheets OAuth2.

8. **Create HTTP Request Node (Token SuiteCRM)**  
   - Connect input from Get Coupon node.  
   - Method: POST  
   - URL: `https://SUITECRMURL/Api/access_token` (replace SUITECRMURL)  
   - Body Parameters:  
     - grant_type: client_credentials  
     - client_id: CLIENTID (replace)  
     - client_secret: CLIENTSECRET (replace)  
   - Allow Unauthorized Certs: true (if needed)  
   - No authentication configured here (using body params).

9. **Create HTTP Request Node (Create Lead SuiteCRM)**  
   - Connect input from Token SuiteCRM node.  
   - Method: POST  
   - URL: `https://SUITECRMURL/Api/V8/module` (replace SUITECRMURL)  
   - Headers:  
     - Authorization: `Bearer {{$node["Token SuiteCRM"].json["access_token"]}}`  
     - Content-Type: `application/vnd.api+json`  
   - Body (JSON):  
     ```json
     {
       "data": {
         "type": "Leads",
         "attributes": {
           "first_name": "={{$node['Form Fields'].json.Name}}",
           "last_name": "={{$node['Form Fields'].json.Surname}}",
           "email1": "={{$node['Form Fields'].json.Email}}",
           "phone_mobile": "={{$node['Form Fields'].json.Phone}}",
           "coupon_c": "={{$node['Get Coupon'].json.COUPON}}"
         }
       }
     }
     ```
   - Send Body and Headers enabled.

10. **Create Google Sheets Node (Update Lead)**  
    - Connect input from Create Lead SuiteCRM node.  
    - Operation: Update row  
    - Document ID and Sheet Name: Same as previous Google Sheets nodes.  
    - Matching Column: "COUPON"  
    - Columns to update:  
      - DATE: `={{ $now.format('dd/LL/yyyy HH:mm:ss') }}`  
      - NAME: `={{ $json.data.attributes.first_name }}`  
      - SURNAME: `={{ $json.data.attributes.last_name }}`  
      - EMAIL: `={{ $json.data.attributes.email1 }}`  
      - PHONE: `={{ $json.data.attributes.phone_mobile }}`  
      - COUPON: `={{ $node['Get Coupon'].json.COUPON }}`  
      - ID LEAD: `={{ $json.data.id }}`  
    - Credentials: Google Sheets OAuth2.

11. **Create Respond to Webhook Node (Respond OK)**  
    - Connect input from Update Lead node.  
    - Response Code: 200  
    - Response Body: `{ "result": "OK", "reason": "lead created" }`

12. **Activate Workflow**  
    - Test by submitting the form with new and duplicate emails.  
    - Verify Google Sheets updates and SuiteCRM lead creation.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Create a Google Sheet with a "COUPON" column pre-filled with unique coupon codes.                    | See sticky note image: https://iili.io/2mXGVwB.md.png and example sheet: https://docs.google.com/spreadsheets/d/1lnRZodxZSOA0QSuzkAb7ZJcfFfNXpX7NcxMdckMTN90/edit?usp=drive_link |
| Workflow supports SuiteCRM versions 7.14.x and 8.x.                                                 | Reminder to create a custom Lead field named 'coupon' in SuiteCRM.                                               |
| SuiteCRM OAuth2 client credentials must be created in SuiteCRM Admin > OAuth2 Client and Token.     | Official SuiteCRM OAuth2 setup guide: https://docs.suitecrm.com/developer/api/developer-setup-guide/json-api/#_generate_private_and_public_key_for_oauth2 |
| Use external forms by hooking their submission to the webhook node and handle responses accordingly.|                                                                                                                  |
| Google Sheets OAuth2 credentials must be configured with access to the target spreadsheet.          |                                                                                                                  |

---

This documentation fully describes the workflow structure, node configurations, logic flow, and setup instructions, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.