🚀 Automated Stripe Payment Recovery: Track Failures & Follow-Up Emails

https://n8nworkflows.xyz/workflows/---automated-stripe-payment-recovery--track-failures---follow-up-emails-5702


# 🚀 Automated Stripe Payment Recovery: Track Failures & Follow-Up Emails

### 1. Workflow Overview

This workflow automates the process of detecting failed Stripe payments and managing follow-up email communications to recover payments. It is designed for businesses using Stripe as their payment processor who want to systematically track payment failures, avoid duplicate notifications, and progressively send recovery emails while tracking email counts.

The workflow is logically divided into two main blocks:

- **1.1 Payment Failure Detection & Logging:** Listens to Stripe webhook events for failed payments, extracts relevant customer and payment information, removes duplicates, and appends or updates this data in a Google Sheet to maintain a centralized log of failed payments.

- **1.2 Scheduled Follow-Up Emailing:** Periodically triggers, retrieves the list of failed payment leads from the Google Sheet, checks how many recovery emails have been sent to each lead, and sends either the first or second recovery email accordingly. It then updates email counts or stops sending emails when appropriate, maintaining the state in the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Payment Failure Detection & Logging

- **Overview:**  
  This block listens for Stripe failed payment events in real-time, processes the event data to extract key user and payment details, ensures no duplicate records, and maintains an updated record in a Google Sheet.

- **Nodes Involved:**  
  - Detect Failed Payments  
  - Extract User and Payment Info  
  - Remove Duplicates  
  - Append or update row in sheet  

- **Node Details:**

  1. **Detect Failed Payments**  
     - *Type & Role:* Stripe Trigger node; listens to Stripe webhook events specifically for failed payments.  
     - *Configuration:* Uses a Stripe webhook with standard configuration to receive failed payment events. No additional parameters set, defaults to catch relevant events.  
     - *Expressions/Variables:* Incoming webhook data contains Stripe event payload.  
     - *Connections:* Output → Extract User and Payment Info  
     - *Version:* v1  
     - *Potential Failures:* Webhook misconfiguration, invalid Stripe credentials, network latency or downtime, unhandled event types.  
     - *Sub-workflows:* None  

  2. **Extract User and Payment Info**  
     - *Type & Role:* Set node; extracts and structures relevant data fields from the Stripe event to prepare for sheet insertion.  
     - *Configuration:* Sets fields such as customer email, payment ID, failure reason, timestamp, and other metadata from the webhook payload using expressions.  
     - *Expressions:* Uses JSON path expressions to map event data to variables.  
     - *Connections:* Input ← Detect Failed Payments; Output → Remove Duplicates  
     - *Version:* v3.4  
     - *Potential Failures:* Expression evaluation errors if Stripe payload schema changes or fields are missing.  
     - *Sub-workflows:* None  

  3. **Remove Duplicates**  
     - *Type & Role:* Remove Duplicates node; ensures no duplicate failed payment records are processed or logged.  
     - *Configuration:* Configured to remove duplicates based on a unique key/field such as payment ID or customer email.  
     - *Connections:* Input ← Extract User and Payment Info; Output → Append or update row in sheet  
     - *Version:* v2  
     - *Potential Failures:* Incorrect duplicate key configuration may lead to missed duplicates or false positives.  
     - *Sub-workflows:* None  

  4. **Append or update row in sheet**  
     - *Type & Role:* Google Sheets node; appends new or updates existing rows in a specific Google Sheet to log failed payment info.  
     - *Configuration:* Points to a Google Sheet document and worksheet dedicated to failed payments; uses credentials with edit access. Uses "append or update" mode keyed by customer email or payment ID to avoid duplicate rows.  
     - *Connections:* Input ← Remove Duplicates  
     - *Version:* v4.6  
     - *Potential Failures:* Google API quota limits, authentication issues, sheet not found, write conflicts.  
     - *Sub-workflows:* None  

---

#### 2.2 Scheduled Follow-Up Emailing

- **Overview:**  
  This block runs on a schedule to retrieve failed payment leads, evaluates how many recovery emails have been sent to each, sends the appropriate follow-up email via SendInBlue, and updates email counts or disables further emailing for leads that have exhausted attempts.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Payment Failure Leads  
  - Check for no. of emails sent  
  - Send First Email  
  - Send Second Email  
  - Update Email Count  
  - Quit Sending Emails to these Leads  

- **Node Details:**

  1. **Schedule Trigger**  
     - *Type & Role:* Schedule Trigger node; initiates the follow-up email process periodically (e.g., daily).  
     - *Configuration:* Configured with a time pattern (unspecified here but likely daily or hourly).  
     - *Connections:* Output → Get Payment Failure Leads  
     - *Version:* v1.2  
     - *Potential Failures:* Time zone misconfiguration, missed triggers if n8n instance is down.  
     - *Sub-workflows:* None  

  2. **Get Payment Failure Leads**  
     - *Type & Role:* Google Sheets node; reads the sheet containing failed payment records to retrieve leads requiring follow-up.  
     - *Configuration:* Reads specific worksheet and columns for customer and email count fields. Requires proper credentials with read access.  
     - *Connections:* Input ← Schedule Trigger; Output → Check for no. of emails sent  
     - *Version:* v4.6  
     - *Potential Failures:* Sheet access errors, empty or malformed data, API limits.  
     - *Sub-workflows:* None  

  3. **Check for no. of emails sent**  
     - *Type & Role:* Switch node; routes leads based on the count of emails previously sent (e.g., 0 or 1).  
     - *Configuration:* Compares an integer field representing the email count; routes 0 to the first email branch, 1 to the second email branch.  
     - *Connections:* Input ← Get Payment Failure Leads; Outputs → Send First Email (for 0), Send Second Email (for 1)  
     - *Version:* v3.2  
     - *Potential Failures:* Incorrect field references, non-integer values, leads with email count >1 (not handled explicitly).  
     - *Sub-workflows:* None  

  4. **Send First Email**  
     - *Type & Role:* SendInBlue node; sends the initial payment recovery email.  
     - *Configuration:* Uses SendInBlue API credentials; email template or content predefined for the first follow-up. Uses customer email from input data.  
     - *Connections:* Input ← Check for no. of emails sent; Output → Update Email Count  
     - *Version:* v1  
     - *Potential Failures:* API authentication errors, email sending failures, invalid email addresses.  
     - *Sub-workflows:* None  

  5. **Send Second Email**  
     - *Type & Role:* SendInBlue node; sends a second, likely more urgent, payment recovery email.  
     - *Configuration:* Similar to first email but with different template/content. Configured to execute once per lead.  
     - *Connections:* Input ← Check for no. of emails sent; Output → Quit Sending Emails to these Leads  
     - *Version:* v1  
     - *Potential Failures:* Same as Send First Email, plus potential logic gap if more than two emails are needed.  
     - *Sub-workflows:* None  

  6. **Update Email Count**  
     - *Type & Role:* Google Sheets node; updates the email count field in the sheet after sending the first email to track progress.  
     - *Configuration:* Points to the same Google Sheet; updates the count cell corresponding to the lead.  
     - *Connections:* Input ← Send First Email  
     - *Version:* v4.6  
     - *Potential Failures:* Write conflicts, API limits, incorrect row targeting.  
     - *Sub-workflows:* None  

  7. **Quit Sending Emails to these Leads**  
     - *Type & Role:* Google Sheets node; updates the sheet to mark leads as no longer to be emailed after the second email is sent.  
     - *Configuration:* Flag field updated to disable further emails or move lead to a “do not contact” list.  
     - *Connections:* Input ← Send Second Email  
     - *Version:* v4.6  
     - *Potential Failures:* Same as Update Email Count.  
     - *Sub-workflows:* None  

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                           | Input Node(s)             | Output Node(s)                   | Sticky Note                                        |
|------------------------------|-------------------------|-----------------------------------------|---------------------------|---------------------------------|---------------------------------------------------|
| Detect Failed Payments        | Stripe Trigger          | Listen for Stripe failed payment events |                           | Extract User and Payment Info   |                                                   |
| Extract User and Payment Info | Set                     | Extract and format payment/user data    | Detect Failed Payments     | Remove Duplicates               |                                                   |
| Remove Duplicates             | Remove Duplicates       | Remove duplicate records                 | Extract User and Payment Info | Append or update row in sheet |                                                   |
| Append or update row in sheet | Google Sheets           | Log failed payment data into Google Sheet | Remove Duplicates         |                                 |                                                   |
| Schedule Trigger             | Schedule Trigger        | Periodic trigger for follow-up emails   |                           | Get Payment Failure Leads       |                                                   |
| Get Payment Failure Leads     | Google Sheets           | Retrieve leads needing follow-up         | Schedule Trigger           | Check for no. of emails sent    |                                                   |
| Check for no. of emails sent  | Switch                  | Route leads based on email count sent    | Get Payment Failure Leads  | Send First Email, Send Second Email |                                                   |
| Send First Email              | SendInBlue              | Send first recovery email                 | Check for no. of emails sent | Update Email Count             |                                                   |
| Update Email Count            | Google Sheets           | Increment email count after first email  | Send First Email           |                                 |                                                   |
| Send Second Email             | SendInBlue              | Send second recovery email                | Check for no. of emails sent | Quit Sending Emails to these Leads |                                                   |
| Quit Sending Emails to these Leads | Google Sheets       | Mark leads as no longer to receive emails | Send Second Email          |                                 |                                                   |
| Sticky Note                  | Sticky Note             | -                                       |                           |                                 |                                                   |
| Sticky Note1                 | Sticky Note             | -                                       |                           |                                 |                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Stripe Trigger Node**  
   - Name: Detect Failed Payments  
   - Type: Stripe Trigger  
   - Configure Stripe webhook for failed payment events with your Stripe credentials.  
   - No additional parameters needed.  

2. **Create Set Node**  
   - Name: Extract User and Payment Info  
   - Type: Set  
   - Add fields to extract from Stripe event payload: customer email, payment ID, failure reason, timestamp, etc., using expressions referencing the Stripe event data.  
   - Connect output from "Detect Failed Payments" to this node.  

3. **Create Remove Duplicates Node**  
   - Name: Remove Duplicates  
   - Type: Remove Duplicates  
   - Configure to remove duplicates based on a unique key such as payment ID or customer email.  
   - Connect output from "Extract User and Payment Info" to this node.  

4. **Create Google Sheets Node**  
   - Name: Append or update row in sheet  
   - Type: Google Sheets  
   - Configure with credentials having write access to your Google Sheet that tracks failed payments.  
   - Set operation to "Append or Update" with unique key as customer email or payment ID.  
   - Connect output from "Remove Duplicates" to this node.  

5. **Create Schedule Trigger Node**  
   - Name: Schedule Trigger  
   - Type: Schedule Trigger  
   - Set the desired recurrence (e.g., daily at a specified time).  

6. **Create Google Sheets Node**  
   - Name: Get Payment Failure Leads  
   - Type: Google Sheets  
   - Configure to read from the same Google Sheet, selecting columns for customer info and email count.  
   - Connect output from "Schedule Trigger" to this node.  

7. **Create Switch Node**  
   - Name: Check for no. of emails sent  
   - Type: Switch  
   - Set condition to check the email count field: route leads with 0 emails sent to first output, leads with 1 email sent to second output.  
   - Connect output from "Get Payment Failure Leads" to this node.  

8. **Create SendInBlue Node (First Email)**  
   - Name: Send First Email  
   - Type: SendInBlue  
   - Configure with SendInBlue credentials and select or define the email template for the first follow-up.  
   - Map recipient email from input data.  
   - Connect first output of "Check for no. of emails sent" to this node.  

9. **Create Google Sheets Node**  
   - Name: Update Email Count  
   - Type: Google Sheets  
   - Configure to update the email count field in the Google Sheet for the lead after sending the first email.  
   - Connect output from "Send First Email" to this node.  

10. **Create SendInBlue Node (Second Email)**  
    - Name: Send Second Email  
    - Type: SendInBlue  
    - Configure similarly with a different template for the second follow-up email.  
    - Connect second output of "Check for no. of emails sent" to this node.  

11. **Create Google Sheets Node**  
    - Name: Quit Sending Emails to these Leads  
    - Type: Google Sheets  
    - Configure to update a flag or status field in the Google Sheet marking the lead as no longer to receive emails after the second email is sent.  
    - Connect output from "Send Second Email" to this node.  

12. **Connect all nodes as per the described flow:**  
    - Detect Failed Payments → Extract User and Payment Info → Remove Duplicates → Append or update row in sheet  
    - Schedule Trigger → Get Payment Failure Leads → Check for no. of emails sent  
      - Check output 0 → Send First Email → Update Email Count  
      - Check output 1 → Send Second Email → Quit Sending Emails to these Leads  

13. **Credential Setup:**  
    - Stripe credentials for Stripe Trigger node  
    - Google OAuth2 credentials with read/write access for Google Sheets nodes  
    - SendInBlue API credentials for both SendInBlue nodes  

14. **Default Values and Constraints:**  
    - Ensure unique keys for duplicate removal and sheet updates  
    - Confirm sheet columns match field mappings exactly  
    - Validate email templates in SendInBlue are tested and active  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                              |
|-------------------------------------------------------------------------------------------------|----------------------------------------------|
| The workflow uses SendInBlue email service for transactional email delivery.                     | SendInBlue API documentation: https://developers.sendinblue.com/ |
| Stripe webhook must be configured in your Stripe Dashboard to point to your n8n Stripe Trigger. | Stripe webhook setup guide: https://stripe.com/docs/webhooks |
| Google Sheets API quotas and permissions must be managed to avoid rate limits.                   | Google Sheets API docs: https://developers.google.com/sheets/api |
| Best practices recommend monitoring logs for webhook failures or email delivery issues.         | n8n community forums and docs for troubleshooting. |

---

*Disclaimer:* The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.