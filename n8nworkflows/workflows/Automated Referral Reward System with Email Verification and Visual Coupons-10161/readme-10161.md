Automated Referral Reward System with Email Verification and Visual Coupons

https://n8nworkflows.xyz/workflows/automated-referral-reward-system-with-email-verification-and-visual-coupons-10161


# Automated Referral Reward System with Email Verification and Visual Coupons

### 1. Workflow Overview

This workflow automates a referral reward system with email verification and personalized coupon generation. It receives referral submissions via a webhook, validates the referrer's email to prevent abuse, then either generates and sends a visual coupon reward or sends a notification about an invalid email. All transactions—both successful and failed—are logged to Google Sheets for further analysis.

**Logical blocks:**

- **1.1 Input Reception:** Webhook node receives referral data from an external form submission.
- **1.2 Email Verification:** Validates the referrer's email address using VerifiEmail API.
- **1.3 Conditional Branch:** Routes workflow based on email verification result.
- **1.4 Coupon Creation:** Builds a personalized HTML coupon template with dynamic data.
- **1.5 HTML to Image Conversion:** Converts the HTML coupon into a PNG image.
- **1.6 Reward Email Delivery:** Sends an email with the coupon image attached to the referrer.
- **1.7 Success Logging:** Logs successful rewards to Google Sheets.
- **1.8 Failure Handling:** Sends invalid email notification and logs failures to Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
Receives referral form submissions via a webhook, capturing critical data like referrer name, email, referred friend's email, and campaign name.

**Nodes Involved:**  
- Webhook Trigger  
- Sticky Note - Webhook (documentation only)

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook (HTTP POST)  
  - Configuration: Path set to `/referral-reward`; listens for POST requests.  
  - Inputs: External HTTP POST requests with JSON payload containing referral data.  
  - Outputs: Passes referral data downstream as JSON.  
  - Edge Cases: Invalid or missing data fields; HTTP errors; malformed JSON.  
  - Notes: Expected payload fields — `referrer_name`, `referrer_email`, `referred_friend`, `campaign`.

- **Sticky Note - Webhook**  
  - Type: Sticky Note (documentation)  
  - Purpose: Describes expected webhook input data and its purpose.

---

#### 1.2 Email Verification

**Overview:**  
Validates the referrer's email address using VerifiEmail API to prevent abuse and fake submissions.

**Nodes Involved:**  
- Verifi Email  
- IF Email Valid?  
- Sticky Note - Email Verification  
- Sticky Note - Conditional Branch

**Node Details:**

- **Verifi Email**  
  - Type: VerifiEmail API node  
  - Configuration: Takes `referrer_email` from webhook JSON, calls VerifiEmail service to validate.  
  - Credentials: Requires VerifiEmail API key.  
  - Outputs: JSON including `valid` boolean field indicating email validity.  
  - Edge Cases: API authentication failure, rate limits, network errors, invalid email format, timeouts.

- **IF Email Valid?**  
  - Type: If node  
  - Configuration: Checks if `{{$json.valid}}` is true.  
  - Inputs: Email verification node output.  
  - Outputs: Two branches—true branch for valid emails, false branch for invalid.  
  - Edge Cases: Missing or malformed `valid` field, expression evaluation errors.

- **Sticky Note - Email Verification**  
  - Type: Sticky Note (documentation)  
  - Purpose: Explains the role of email validation to prevent invalid referrals.

- **Sticky Note - Conditional Branch**  
  - Type: Sticky Note (documentation)  
  - Purpose: Describes the decision logic based on email validity.

---

#### 1.3 Conditional Branch

**Overview:**  
Routes the workflow into two branches depending on email validity: coupon generation for valid emails or sending a failure notice for invalid emails.

**Nodes Involved:**  
- IF Email Valid? (from previous block)  
- Set Coupon Template (true branch)  
- Send Invalid Email Notice (false branch)  
- Sticky Note - Conditional Branch (previously mentioned)

**Node Details:**

- **Set Coupon Template**  
  - Type: Set node  
  - Configuration: Creates variables for coupon generation: `referrer_name`, `referrer_email`, `campaign` from webhook, and dynamically generates `coupon_code` with format like `REF-JOHN1234`.  
  - Inputs: Valid email branch from IF node.  
  - Outputs: JSON with coupon template variables for next block.  
  - Edge Cases: Expression errors generating coupon code; missing input fields.

- **Send Invalid Email Notice**  
  - Type: Gmail node (send email)  
  - Configuration: Sends an email to admin notifying of failed email verification; includes a user-friendly explanation and instructions.  
  - Credentials: Requires Gmail OAuth2 credentials.  
  - Inputs: Invalid email branch from IF node, uses webhook data for personalization.  
  - Edge Cases: SMTP errors, authentication failure, invalid recipient email.

- **Sticky Note - Conditional Branch**  
  - Provides context for the conditional logic and branches.

---

#### 1.4 Coupon Creation

**Overview:**  
Generates a personalized HTML coupon card with dynamic content for the referrer.

**Nodes Involved:**  
- Set Coupon Template (from previous block)  
- Sticky Note - Coupon Template

**Node Details:**

- **Set Coupon Template**  
  - Already detailed above; creates coupon data.

- **Sticky Note - Coupon Template**  
  - Describes coupon contents: HTML/CSS card with dynamic fields like referrer name, coupon code, and campaign.

---

#### 1.5 HTML to Image Conversion

**Overview:**  
Converts the HTML coupon template into a high-quality PNG image using the HTMLCSStoImage API.

**Nodes Involved:**  
- HTML/CSS to Image  
- Sticky Note - HTML to Image

**Node Details:**

- **HTML/CSS to Image**  
  - Type: HTMLCSStoImage node  
  - Configuration: Uses a detailed HTML template with embedded CSS and dynamic placeholders for referrer name, coupon code, campaign, etc.  
  - Credentials: Requires HTMLCSStoImage API user ID and key.  
  - Inputs: Coupon data from Set Coupon Template node.  
  - Outputs: JSON containing `image_url` of generated PNG coupon.  
  - Edge Cases: API request failure, invalid HTML, rate limits, network errors.

- **Sticky Note - HTML to Image**  
  - Explains the purpose of converting HTML coupon to image.

---

#### 1.6 Reward Email Delivery

**Overview:**  
Sends a personalized reward email to the referrer including the coupon image and details.

**Nodes Involved:**  
- Send Reward Email (Gmail)  
- Sticky Note - Reward Email

**Node Details:**

- **Send Reward Email (Gmail)**  
  - Type: Gmail node (send email)  
  - Configuration: HTML email body with personalized greeting, embedded coupon image (`image_url`), coupon code, validity info (30 days), and campaign details.  
  - Credentials: Gmail OAuth2 required.  
  - Inputs: Image URL and coupon data from HTML/CSS to Image node and Set Coupon Template node.  
  - Outputs: Passes to logging node.  
  - Edge Cases: SMTP errors, invalid email addresses, authentication issues.

- **Sticky Note - Reward Email**  
  - Describes email content structure and purpose.

---

#### 1.7 Success Logging

**Overview:**  
Logs details of successful referral rewards to a Google Sheet for tracking and analytics.

**Nodes Involved:**  
- Log to Google Sheets (Success)  
- Sticky Note - Log Success

**Node Details:**

- **Log to Google Sheets (Success)**  
  - Type: Google Sheets node  
  - Configuration: Appends or updates a row with columns: Timestamp, Referrer Name, Referrer Email, Status (“Reward Sent”), Coupon Code, Coupon Image URL, Campaign.  
  - Credentials: Google Sheets OAuth2 required.  
  - Inputs: Data from Send Reward Email node and coupon info.  
  - Edge Cases: API rate limits, authentication failures, sheet not found, permission issues.

- **Sticky Note - Log Success**  
  - Explains tracking purpose and required sheet setup.

---

#### 1.8 Failure Handling

**Overview:**  
Sends an invalid email notification and logs failures to Google Sheets to track issues and improve system robustness.

**Nodes Involved:**  
- Send Invalid Email Notice (from 1.3)  
- Log to Google Sheets (Failed)  
- Sticky Note - Invalid Email Notice  
- Sticky Note - Log Failed

**Node Details:**

- **Send Invalid Email Notice**  
  - Already detailed above; sends polite failure notification to admin.

- **Log to Google Sheets (Failed)**  
  - Type: Google Sheets node  
  - Configuration: Logs failed verification attempts with Timestamp, Referrer Name, Referrer Email, Status (“Email Verification Failed”), and Campaign. Coupon columns omitted.  
  - Credentials: Google Sheets OAuth2 required.  
  - Inputs: From Send Invalid Email Notice node.  
  - Edge Cases: Same as success logging node.

- **Sticky Note - Invalid Email Notice**  
  - Details email content and rationale.

- **Sticky Note - Log Failed**  
  - Describes analytics value of tracking failed attempts.

---

### 3. Summary Table

| Node Name                 | Node Type                   | Functional Role                              | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                  |
|---------------------------|-----------------------------|----------------------------------------------|-----------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note - Setup Credentials | Sticky Note                | Credentials setup instructions                | —                           | —                                 | 🔐 SETUP CREDENTIALS REQUIRED: VerifiEmail, HTMLCSStoImage, Gmail 2FA, Google Sheets setup.  |
| Webhook Trigger           | Webhook                     | Receives referral form submissions             | —                           | Verifi Email                      | 🎯 WEBHOOK TRIGGER: Receives referral data with fields: referrer_name, referrer_email, etc.   |
| Sticky Note - Webhook     | Sticky Note                 | Describes webhook input data                   | —                           | —                                 | See above.                                                                                   |
| Verifi Email              | VerifiEmail API             | Validates referrer email                        | Webhook Trigger             | IF Email Valid?                   | ✅ EMAIL VERIFICATION: Validates email to prevent abuse                                     |
| IF Email Valid?           | If                          | Routes workflow based on email validity        | Verifi Email                | Set Coupon Template (true branch), Send Invalid Email Notice (false branch) | ⚖️ CONDITIONAL LOGIC: TRUE → coupon; FALSE → invalid email notification                     |
| Sticky Note - Email Verification | Sticky Note                 | Explains email verification process            | —                           | —                                 | See above.                                                                                   |
| Sticky Note - Conditional Branch | Sticky Note                 | Explains conditional routing of workflow       | —                           | —                                 | See above.                                                                                   |
| Set Coupon Template       | Set                         | Creates coupon data and unique coupon code     | IF Email Valid? (true)      | HTML/CSS to Image                 | 🎨 COUPON TEMPLATE CREATION: Dynamic HTML coupon fields including coupon code                |
| Sticky Note - Coupon Template | Sticky Note                 | Describes coupon template content               | —                           | —                                 | See above.                                                                                   |
| HTML/CSS to Image         | HTMLCSStoImage              | Converts HTML coupon to PNG image               | Set Coupon Template         | Send Reward Email (Gmail)         | 🖼️ HTML TO IMAGE CONVERSION: Converts dynamic HTML coupon to image                          |
| Sticky Note - HTML to Image | Sticky Note                 | Explains HTML to image conversion                | —                           | —                                 | See above.                                                                                   |
| Send Reward Email (Gmail) | Gmail                       | Sends personalized reward email with coupon    | HTML/CSS to Image           | Log to Google Sheets (Success)    | 📧 SEND REWARD EMAIL: Sends email with embedded coupon image and details                   |
| Sticky Note - Reward Email| Sticky Note                 | Describes reward email content                   | —                           | —                                 | See above.                                                                                   |
| Log to Google Sheets (Success) | Google Sheets               | Logs successful referral rewards                 | Send Reward Email (Gmail)   | —                                 | 📊 LOG TO GOOGLE SHEETS (SUCCESS): Tracks all successful rewards                          |
| Sticky Note - Log Success | Sticky Note                 | Explains logging success to Google Sheets       | —                           | —                                 | See above.                                                                                   |
| Send Invalid Email Notice | Gmail                       | Sends notification about invalid email submission| IF Email Valid? (false)      | Log to Google Sheets (Failed)     | ❌ INVALID EMAIL NOTIFICATION: Polite failure notice with reasons and instructions         |
| Sticky Note - Invalid Email Notice | Sticky Note                 | Explains invalid email notice email content      | —                           | —                                 | See above.                                                                                   |
| Log to Google Sheets (Failed) | Google Sheets               | Logs failed email verification attempts          | Send Invalid Email Notice   | —                                 | 📊 LOG TO GOOGLE SHEETS (FAILED): Tracks failed verifications for analytics                |
| Sticky Note - Log Failed  | Sticky Note                 | Explains analytics value of failed logs          | —                           | —                                 | See above.                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `referral-reward`  
   - Purpose: Receive referral submissions with JSON payload: `referrer_name`, `referrer_email`, `referred_friend`, `campaign`.

2. **Add Verifi Email node**  
   - Type: VerifiEmail API node  
   - Credentials: Setup with VerifiEmail API key (sign up at https://verifi.email)  
   - Input: Use `{{$json.body.referrer_email}}` from webhook trigger  
   - Purpose: Validate the referrer’s email address.

3. **Add an If node (IF Email Valid?)**  
   - Condition: Boolean check if `{{$json.valid}}` equals `true`  
   - Inputs: Verifi Email node output  
   - Outputs:  
     - True branch: proceed to coupon generation  
     - False branch: proceed to invalid email handling.

4. **On True branch, add a Set node (Set Coupon Template)**  
   - Fields to set:  
     - `referrer_name`: `{{$json.body.referrer_name}}`  
     - `referrer_email`: `{{$json.body.referrer_email}}`  
     - `campaign`: `{{$json.body.campaign}}`  
     - `coupon_code`: Generate with expression like: `=REF-{{ $json.body.referrer_name.toUpperCase().slice(0,4) }}{{ Math.floor(Math.random()*9999) }}`  
   - Purpose: Prepare data for coupon creation.

5. **Add HTML/CSS to Image node**  
   - Credentials: Setup with HTMLCSStoImage API user ID and key (https://htmlcsstoimg.com)  
   - Input HTML: Use a well-styled HTML template that references variables from Set Coupon Template node, e.g., referrer name, coupon code, campaign.  
   - Purpose: Convert the coupon HTML into a PNG image.

6. **Add Gmail node (Send Reward Email)**  
   - Credentials: Gmail OAuth2 with 2FA enabled  
   - Send To: `{{$json.referrer_email}}` from Set Coupon Template  
   - Subject: “🎁 Your Referral Reward Coupon is Here!”  
   - Message Body: HTML formatted email embedding the PNG image URL from the HTML/CSS to Image node, including coupon code and campaign details.  
   - Purpose: Deliver the reward email.

7. **Add Google Sheets node (Log to Google Sheets - Success)**  
   - Credentials: Google Sheets OAuth2  
   - Document ID and Sheet Name: Setup with your Google Sheets document and sheet titled `Referral_Reward_Tracker`  
   - Operation: Append or update row with columns: Timestamp, Referrer Name, Referrer Email, Status (“Reward Sent”), Coupon Code, Coupon Image URL, Campaign.  
   - Inputs: Data from Send Reward Email node and coupon details.

8. **On False branch of IF node (invalid email): Add Gmail node (Send Invalid Email Notice)**  
   - Credentials: Gmail OAuth2  
   - Send To: Admin email (e.g., adminexample@gmail.com)  
   - Subject: “Referral Submission - Email Verification Failed”  
   - Message Body: Polite HTML email explaining failure reasons and instructions to resubmit.

9. **Add Google Sheets node (Log to Google Sheets - Failed)**  
   - Credentials: Google Sheets OAuth2  
   - Document ID and Sheet Name: Same as above  
   - Operation: Append or update row with columns: Timestamp, Referrer Name, Referrer Email, Status (“Email Verification Failed”), Campaign.  
   - Inputs: Data from Send Invalid Email Notice node.

10. **Connect nodes according to success and failure branches:**  
    - Webhook Trigger → Verifi Email → IF Email Valid?  
    - IF Email Valid? (true) → Set Coupon Template → HTML/CSS to Image → Send Reward Email → Log to Google Sheets (Success)  
    - IF Email Valid? (false) → Send Invalid Email Notice → Log to Google Sheets (Failed)

11. **Test each credential connection:**  
    - VerifiEmail API key  
    - HTMLCSStoImage API key and User ID  
    - Gmail OAuth2 with 2FA enabled  
    - Google Sheets OAuth2 with proper access to the spreadsheet

12. **Create the Google Sheet:**  
    - Sheet name: `Referral_Reward_Tracker`  
    - Headers: `Timestamp | Referrer Name | Referrer Email | Status | Coupon Code | Coupon Image URL | Campaign`

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow requires setting up multiple third-party credentials (VerifiEmail, HTMLCSStoImage, Gmail OAuth2, Google Sheets OAuth2).      | Credential setup instructions detailed in sticky note “🔐 SETUP CREDENTIALS REQUIRED”. |
| Gmail account used must have 2FA enabled for OAuth2 authentication to work securely.                                                  | Gmail OAuth2 credential setup                    |
| Google Sheet must be shared with the OAuth2 service account or user to allow editing.                                                  | Google Sheets integration                        |
| Coupon HTML template includes modern CSS styling with gradients, shadows, and responsive layout for aesthetic appeal.               | See HTML in HTML/CSS to Image node parameters   |
| Email notifications (both reward and failure) use HTML formatting for better user experience.                                         | Gmail node message parameters                     |
| Tracking in Google Sheets supports analytics to identify referral program effectiveness and potential fraud patterns.               | Sticky notes on logging nodes                     |
| For API key management and service signups, refer to: https://verifi.email and https://htmlcsstoimg.com                              | External service signup links                      |

---

**Disclaimer:** The content provided is extracted solely from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.