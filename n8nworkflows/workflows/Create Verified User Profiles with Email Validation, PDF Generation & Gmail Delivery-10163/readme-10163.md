Create Verified User Profiles with Email Validation, PDF Generation & Gmail Delivery

https://n8nworkflows.xyz/workflows/create-verified-user-profiles-with-email-validation--pdf-generation---gmail-delivery-10163


# Create Verified User Profiles with Email Validation, PDF Generation & Gmail Delivery

### 1. Workflow Overview

This workflow automates the creation of verified user profiles by validating user email addresses, generating a personalized PDF profile, and delivering it via email. It targets scenarios such as user registrations or signups where email verification and professional profile creation are required. The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Receives user data via HTTP POST webhook.
- **1.2 Email Verification:** Validates the user's email using the VerifiEmail API.
- **1.3 Conditional Branching:** Routes flow based on email validity.
- **1.4 Profile Generation:** Creates an HTML profile template dynamically and converts it to PDF.
- **1.5 PDF Download:** Retrieves the generated PDF as a binary file.
- **1.6 Email Delivery:** Sends the PDF as an email attachment via Gmail.
- **1.7 Data Logging:** Logs verified user details into a Google Sheets document.
- **1.8 Invalid Email Handling:** Sends a rejection email if verification fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives user signup data through an HTTP POST request to a dedicated webhook URL. Expected JSON includes user details like name, email, city, profession, and bio.

- **Nodes Involved:**  
  - Webhook - Receive User Data

- **Node Details:**  
  - **Webhook - Receive User Data**  
    - *Type:* Webhook  
    - *Role:* Entry point to receive user data via POST at path "/create-profile".  
    - *Configuration:* HTTP Method set to POST; responseMode set to "lastNode" to send response from last executed node.  
    - *Key Expressions:* Accesses incoming JSON body fields, e.g., `{{$json.body.email}}`.  
    - *Connections:* Output connected to "Verifi Email" node.  
    - *Edge Cases:* Malformed requests, missing fields, or unsupported HTTP methods may cause failure or no workflow trigger.

#### 1.2 Email Verification

- **Overview:**  
Validates the provided email address using the VerifiEmail API to determine if the email is valid and trustworthy.

- **Nodes Involved:**  
  - Verifi Email

- **Node Details:**  
  - **Verifi Email**  
    - *Type:* VerifiEmail API node  
    - *Role:* Validates email address authenticity.  
    - *Configuration:* Uses the email extracted from webhook data (`{{$json.body.email}}`).  
    - *Credentials:* Requires VerifiEmail API key.  
    - *Connections:* Outputs to "IF Email Valid?" node.  
    - *Edge Cases:* API key misconfiguration, network issues, or invalid email format may cause errors or false negatives.

#### 1.3 Conditional Branching

- **Overview:**  
Routes the workflow based on the result of the email verification. If valid, proceeds to profile generation; if invalid, sends rejection email.

- **Nodes Involved:**  
  - IF Email Valid?

- **Node Details:**  
  - **IF Email Valid?**  
    - *Type:* IF node (boolean condition)  
    - *Role:* Checks whether the "valid" property returned by VerifiEmail is true.  
    - *Condition:* `{{$json.valid === true}}`  
    - *Connections:*  
      - True branch → "Generate HTML Template"  
      - False branch → "Send Rejection Email - Gmail"  
    - *Edge Cases:* If the API response structure changes, expression may fail; missing "valid" property leads to false branch.

#### 1.4 Profile Generation

- **Overview:**  
Generates a styled HTML profile using user data, then converts the HTML into a PDF via an external service.

- **Nodes Involved:**  
  - Generate HTML Template  
  - HTML to PDF

- **Node Details:**  
  - **Generate HTML Template**  
    - *Type:* Set node  
    - *Role:* Constructs an HTML string with inline CSS styling, dynamically injecting user data from webhook node.  
    - *Configuration:* Uses template expressions to embed `name`, `email`, `city`, `profession`, and `bio`.  
    - *Connections:* Output to "HTML to PDF".  
    - *Edge Cases:* Missing user data fields may cause incomplete profiles; expression errors in template can break HTML output.

  - **HTML to PDF**  
    - *Type:* HTMLcsstoPDF API node  
    - *Role:* Converts the generated HTML content into a PDF file URL.  
    - *Configuration:* Passes HTML content from previous node.  
    - *Credentials:* Requires HTMLcsstoPDF API keys (User ID & API Key).  
    - *Connections:* Output (PDF URL) to "Download PDF Binary".  
    - *Edge Cases:* API limits, service downtime, or invalid HTML input can cause failure or malformed PDF.

#### 1.5 PDF Download

- **Overview:**  
Downloads the generated PDF file as binary data from the URL provided by the PDF generation API.

- **Nodes Involved:**  
  - Download PDF Binary

- **Node Details:**  
  - **Download PDF Binary**  
    - *Type:* HTTP Request node  
    - *Role:* Performs GET request to fetch PDF binary data.  
    - *Configuration:* URL set dynamically from `{{$json.pdf_url}}`, response format set to "file" for binary.  
    - *Connections:* Output to "Send Profile Email - Gmail".  
    - *Edge Cases:* Broken URL, network issues, or slow responses can cause timeouts or failures.

#### 1.6 Email Delivery

- **Overview:**  
Sends a personalized email to the verified user with the PDF profile attached, using Gmail via OAuth2 authentication.

- **Nodes Involved:**  
  - Send Profile Email - Gmail

- **Node Details:**  
  - **Send Profile Email - Gmail**  
    - *Type:* Gmail node  
    - *Role:* Sends an HTML email with an attached PDF profile.  
    - *Configuration:*  
      - Recipient email: dynamic from webhook data.  
      - Subject: "Your Verified Profile PDF is Ready ✅".  
      - Message body: Rich HTML content with user details and professional styling.  
      - Attachment: PDF binary from previous node.  
    - *Credentials:* Gmail OAuth2 credentials required.  
    - *Connections:* Output to "Log to Google Sheets".  
    - *Edge Cases:* OAuth token expiry, Gmail send limits, or attachment size limits may cause errors.

#### 1.7 Data Logging

- **Overview:**  
Records the verified user profile data into a Google Sheets spreadsheet for audit and tracking.

- **Nodes Involved:**  
  - Log to Google Sheets

- **Node Details:**  
  - **Log to Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Appends or updates a row with user data in a specified sheet.  
    - *Configuration:*  
      - Columns include Name, Email, City, Profession, Verified status (checkmark), Date.  
      - Matching column: Email (to update existing entries).  
      - Spreadsheet and sheet identified by document ID and gid.  
    - *Credentials:* Google Sheets OAuth2, same Google account as Gmail.  
    - *Connections:* Final node on success path.  
    - *Edge Cases:* OAuth permission issues, sheet access rights, or schema mismatch may cause failures.

#### 1.8 Invalid Email Handling

- **Overview:**  
Sends a rejection email to users whose email validation fails, explaining possible reasons and next steps.

- **Nodes Involved:**  
  - Send Rejection Email - Gmail

- **Node Details:**  
  - **Send Rejection Email - Gmail**  
    - *Type:* Gmail node  
    - *Role:* Sends an HTML email notifying verification failure.  
    - *Configuration:*  
      - Recipient: email from webhook.  
      - Subject: "Profile Verification Failed ❌".  
      - Body: Explanation of common failure causes and guidance.  
    - *Credentials:* Gmail OAuth2 credentials.  
    - *Connections:* Terminal node on failure path.  
    - *Edge Cases:* Same as above for Gmail node; invalid recipient email might cause send failure.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                         | Input Node(s)                 | Output Node(s)                            | Sticky Note                                                                                              |
|----------------------------|---------------------------------|---------------------------------------|------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Sticky Note1               | Sticky Note                     | Setup credentials information          |                              |                                           | ## 🔧 SETUP CREDENTIALS Required Credentials: VerifiEmail API, HTMLcsstoPDF API, Gmail OAuth2, Google Sheets OAuth2 |
| Webhook - Receive User Data | Webhook                        | Receive user signup data via POST      |                              | Verifi Email                             | ## 📥 STEP 1: WEBHOOK TRIGGER Expected JSON input example provided                                      |
| Sticky Note2               | Sticky Note                     | Describes webhook input format         |                              |                                           | ## 📥 STEP 1: WEBHOOK TRIGGER Expected JSON input example provided                                      |
| Verifi Email               | VerifiEmail API Node            | Validate user email                     | Webhook - Receive User Data   | IF Email Valid?                          | ## ✅ STEP 2: EMAIL VERIFICATION Email validated via VerifiEmail API                                    |
| Sticky Note3               | Sticky Note                     | Email verification purpose description |                              |                                           | ## ✅ STEP 2: EMAIL VERIFICATION Email validated via VerifiEmail API                                    |
| IF Email Valid?            | IF                             | Branch based on email verification result | Verifi Email                | Generate HTML Template (true), Send Rejection Email - Gmail (false) | ## ⚖️ STEP 3: CONDITIONAL LOGIC Describes true/false paths                                            |
| Sticky Note4               | Sticky Note                     | Explains conditional branching         |                              |                                           | ## ⚖️ STEP 3: CONDITIONAL LOGIC Describes true/false paths                                            |
| Generate HTML Template     | Set                            | Create HTML profile template            | IF Email Valid? (true)        | HTML to PDF                             | ## 🧾 STEP 4: HTML TEMPLATE CREATION Modern CSS styled, dynamic data                                  |
| Sticky Note5               | Sticky Note                     | Describes HTML template creation        |                              |                                           | ## 🧾 STEP 4: HTML TEMPLATE CREATION Modern CSS styled, dynamic data                                  |
| HTML to PDF                | HTMLcsstoPDF API Node           | Convert HTML to PDF                     | Generate HTML Template        | Download PDF Binary                     | ## 📄 STEP 5: PDF GENERATION Converts HTML to PDF/image with font and viewport settings               |
| Sticky Note6               | Sticky Note                     | Describes PDF generation                |                              |                                           | ## 📄 STEP 5: PDF GENERATION Converts HTML to PDF/image with font and viewport settings               |
| Download PDF Binary        | HTTP Request                   | Download generated PDF as binary        | HTML to PDF                   | Send Profile Email - Gmail             | ## 💾 STEP 6: DOWNLOAD PDF Downloads PDF binary data from generated URL                               |
| Sticky Note7               | Sticky Note                     | Describes PDF download step             |                              |                                           | ## 💾 STEP 6: DOWNLOAD PDF Downloads PDF binary data from generated URL                               |
| Send Profile Email - Gmail | Gmail                         | Email verified profile with PDF attached | Download PDF Binary          | Log to Google Sheets                   | ## 📧 STEP 7: EMAIL DELIVERY Sends PDF profile via Gmail with rich HTML email                         |
| Sticky Note8               | Sticky Note                     | Describes email delivery                 |                              |                                           | ## 📧 STEP 7: EMAIL DELIVERY Sends PDF profile via Gmail with rich HTML email                         |
| Log to Google Sheets       | Google Sheets                  | Log verified user data                   | Send Profile Email - Gmail    |                                           | ## 📊 STEP 8: DATA LOGGING Logs user data with verification status in Google Sheets                   |
| Sticky Note9               | Sticky Note                     | Describes data logging in sheets         |                              |                                           | ## 📊 STEP 8: DATA LOGGING Logs user data with verification status in Google Sheets                   |
| Send Rejection Email - Gmail | Gmail                       | Notify user of failed email verification | IF Email Valid? (false)        |                                           | ## 🚫 STEP 9: INVALID EMAIL HANDLING Sends rejection email with reasons and next steps                |
| Sticky Note10              | Sticky Note                     | Describes invalid email handling         |                              |                                           | ## 🚫 STEP 9: INVALID EMAIL HANDLING Sends rejection email with reasons and next steps                |
| Sticky Note11              | Sticky Note                     | Workflow completion summary              |                              |                                           | ## ✅ WORKFLOW COMPLETE! Summarizes success and failure paths                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `create-profile`  
   - HTTP Method: POST  
   - Response Mode: Last Node  
   - Purpose: Receive user JSON data with fields: name, email, city, profession, bio.

2. **Create VerifiEmail Node**  
   - Type: VerifiEmail API node  
   - Input: Use expression `{{$json.body.email}}` to pass email from webhook.  
   - Credential: Connect your VerifiEmail API credentials (API key).  
   - Output: Connect to IF node.

3. **Create IF Node**  
   - Type: IF node (Boolean)  
   - Condition: Check if `{{$json.valid}}` is true.  
   - True branch: Connect to HTML template generation.  
   - False branch: Connect to rejection email node.

4. **Create "Generate HTML Template" Set Node**  
   - Type: Set node  
   - Add a string field named `html_content`.  
   - Use a comprehensive HTML template with CSS styling embedding webhook data via expressions like `{{ $('Webhook - Receive User Data').item.json.body.name }}` for dynamic content.  
   - Connect output to HTML to PDF node.

5. **Create HTML to PDF Node**  
   - Type: HTMLcsstoPDF API node  
   - Input: Use expression `{{$json.html_content}}` for HTML content.  
   - Credential: Provide HTMLcsstoPDF User ID and API Key.  
   - Output: Connect to HTTP Request node to download PDF.

6. **Create Download PDF Binary HTTP Request Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression `{{$json.pdf_url}}` from previous node's output.  
   - Response Format: File (binary).  
   - Connect to Gmail send email node.

7. **Create Send Profile Email Gmail Node**  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 credentials.  
   - To: Use expression `{{$('Webhook - Receive User Data').item.json.body.email}}`.  
   - Subject: "Your Verified Profile PDF is Ready ✅"  
   - Message: Use a styled HTML email with embedded user data and personalization.  
   - Attachments: Attach binary data from the PDF download node.  
   - Connect output to Google Sheets node.

8. **Create Log to Google Sheets Node**  
   - Type: Google Sheets  
   - Credential: Google Sheets OAuth2 credentials (same account as Gmail).  
   - Operation: Append or Update row.  
   - Document ID: Your Google Sheets document ID.  
   - Sheet Name: Sheet1 (or your target sheet).  
   - Columns: Map Name, Email, City, Profession, Verified (set to "✅"), Date (current date).  
   - Matching Column: Email.  
   - Connect as final node in success path.

9. **Create Send Rejection Email Gmail Node**  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 credentials.  
   - To: Use expression `{{$('Webhook - Receive User Data').item.json.body.email}}`.  
   - Subject: "Profile Verification Failed ❌"  
   - Message: HTML content explaining failure reasons and next steps.  
   - No further connections (terminal node for failure).

10. **Add Sticky Notes**  
    - Add informative sticky notes at relevant places describing credential setup, each logical step, input/output expectations, and workflow summary as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Credential Setup: VerifiEmail API (https://verifi.email), HTMLcsstoPDF API (https://htmlcsstoimg.com), Gmail OAuth2, Google Sheets OAuth2 | Sticky Note1                                                     |
| Expected Webhook JSON Input Format Example                                                                    | Sticky Note2                                                     |
| Email Verification Purpose and Parameters                                                                     | Sticky Note3                                                     |
| Workflow Branching Logic Description                                                                           | Sticky Note4                                                     |
| HTML Template Styling and Features                                                                              | Sticky Note5                                                     |
| PDF Generation Request Details                                                                                   | Sticky Note6                                                     |
| PDF Download Explanation                                                                                         | Sticky Note7                                                     |
| Email Delivery Configuration Details                                                                              | Sticky Note8                                                     |
| Data Logging in Google Sheets Column Details                                                                     | Sticky Note9                                                     |
| Invalid Email Handling Explanation and Common Failure Reasons                                                     | Sticky Note10                                                    |
| Workflow Completion Summary                                                                                        | Sticky Note11                                                    |

---

**Disclaimer:**  
The provided text is exclusively sourced from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are lawful and public.