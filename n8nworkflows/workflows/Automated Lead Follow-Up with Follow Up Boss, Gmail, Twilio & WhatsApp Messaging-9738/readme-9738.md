Automated Lead Follow-Up with Follow Up Boss, Gmail, Twilio & WhatsApp Messaging

https://n8nworkflows.xyz/workflows/automated-lead-follow-up-with-follow-up-boss--gmail--twilio---whatsapp-messaging-9738


# Automated Lead Follow-Up with Follow Up Boss, Gmail, Twilio & WhatsApp Messaging

### 1. Workflow Overview

This workflow automates a lead follow-up process by integrating Follow Up Boss (FUB) CRM with Gmail, Twilio SMS/WhatsApp messaging, and n8n automation. It is designed to:

- Regularly fetch new leads from Follow Up Boss since the last execution.
- Validate and filter leads based on the validity of their email and phone number.
- Route leads into three distinct paths according to data quality:
  - Leads with both valid email and phone receive both email and SMS/WhatsApp messages.
  - Leads with invalid or missing phone only receive emails.
  - Leads with invalid or missing email only receive SMS/WhatsApp messages.
- Personalize and send follow-up messages via Gmail and Twilio.
- Record the last run timestamp to ensure incremental processing and avoid duplicates.

The workflow divides into four main logical blocks:

- **1.1 Scheduled Trigger & Lead Retrieval:** Periodically fetch latest leads from Follow Up Boss since the last run.
- **1.2 Lead Validation & Filtering:** Split leads and filter based on email and phone validity.
- **1.3 Follow-Up Branching Logic:** Route leads into appropriate messaging channels (email, SMS/WhatsApp, or both).
- **1.4 Message Personalization & Dispatch:** Extract relevant fields, send messages, and update the last run timestamp.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Lead Retrieval

**Overview:**  
This block triggers the workflow every 3 minutes, retrieves the timestamp of the last run, and fetches new leads created after this time from Follow Up Boss.

**Nodes Involved:**  
- Schedule Trigger  
- Get Last time run (Code)  
- Get Last Lead (FUB) (HTTP Request)  
- Wait  
- DIvide Each Lead (Split Out)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 3 minutes.  
  - Configuration: Interval set to 3 minutes.  
  - Input: None (start node)  
  - Output: Triggers "Get Last time run"  
  - Potential Failures: Scheduling misconfiguration; workflow disabled.

- **Get Last time run**  
  - Type: Code (JavaScript)  
  - Role: Retrieves the ISO timestamp of the last workflow execution from static global data. Defaults to 15 minutes ago if no prior run.  
  - Code extracts `lastRun` from workflow static data or sets a fallback timestamp.  
  - Input: Trigger from Schedule Trigger  
  - Output: Passes `lastRun` timestamp to "Get Last Lead (FUB)."  
  - Edge Cases: Static data corruption; time zone mismatches.

- **Get Last Lead (FUB)**  
  - Type: HTTP Request  
  - Role: Calls Follow Up Boss API to fetch leads created after `lastRun`.  
  - Configuration: HTTP Basic Auth credential; GET request to `https://api.followupboss.com/v1/people?createdAfter={{$json["lastRun"]}}`.  
  - Input: Receives `lastRun` from code node.  
  - Output: Passes lead data array to "Wait" node.  
  - Edge Cases: API auth failures, rate limiting, no new leads.

- **Wait**  
  - Type: Wait  
  - Role: Adds a fixed 6-second delay before processing (possibly to handle API rate limits or data consistency).  
  - Input: Leads array from HTTP Request node.  
  - Output: Passes to "DIvide Each Lead."  
  - Edge Cases: Unnecessary delays if no leads; workflow slowdown.

- **DIvide Each Lead (Split Out)**  
  - Type: Split Out  
  - Role: Splits the array of leads into individual lead items for separate processing.  
  - Field to split: `people` array from FUB response.  
  - Input: From Wait node.  
  - Output: Each lead item to "Invalid Number and Email" node.  
  - Edge Cases: Empty arrays; malformed data structure.

---

#### 1.2 Lead Validation & Filtering

**Overview:**  
Validates each lead’s email and phone status, filters invalid or partially valid entries and routes leads accordingly.

**Nodes Involved:**  
- Invalid Number and Email (Filter)  
- All Good? (If)  
- False Email or False Phone (If)  
- Sticky Note (contextual explanation)

**Node Details:**

- **Invalid Number and Email**  
  - Type: Filter  
  - Role: Filters leads that have either valid email or valid phone.  
  - Conditions: Passes leads where email status equals "Valid" OR phone status equals "Valid."  
  - Input: Single lead items.  
  - Output: Leads with at least one valid contact method to "All Good?" node.  
  - Edge Cases: Leads with no valid contact data filtered out.

- **All Good?**  
  - Type: If  
  - Role: Checks if both email and phone statuses are "Valid."  
  - Condition: Email status == "Valid" AND phone status == "Valid".  
  - Input: Leads passing from previous filter.  
  - Output:  
    - True: Leads with both valid email and phone to "Full data → Send both Email + SMS/WhatsApp" branch.  
    - False: Leads with either invalid email or phone to "False Email or False Phone."  
  - Edge Cases: Null or missing status fields causing condition failure.

- **False Email or False Phone**  
  - Type: If  
  - Role: Further splits leads with invalid contact info into two paths: missing/invalid phone or missing/invalid email.  
  - Input: Leads with one invalid contact method.  
  - Output:  
    - True: Leads missing/invalid phone to "Missing/Invalid phone → Email only."  
    - False: Leads missing/invalid email to "Missing/Invalid email → SMS/WhatsApp only."  
  - Edge Cases: Leads missing both contacts may not be handled explicitly.

- **Sticky Note** (🟦 STEP 2: Validation & Filtering)  
  - Content: Explains validation and filtering purpose to avoid invalid or duplicate messages.

---

#### 1.3 Follow-Up Branching Logic

**Overview:**  
Based on validation, leads are routed to three separate batch processing nodes to manage sending of personalized messages via email and/or SMS/WhatsApp.

**Nodes Involved:**  
- Full data → Send both Email + SMS/WhatsApp (Split In Batches)  
- Missing/Invalid phone → Email only (Split In Batches)  
- Missing/Invalid email → SMS/WhatsApp only (Split In Batches)  
- Sticky Note (🟨 STEP 3: Smart Follow-Up Logic)

**Node Details:**

- **Full data → Send both Email + SMS/WhatsApp**  
  - Type: Split In Batches  
  - Role: Processes leads with full valid contact info in manageable batches for messaging.  
  - Input: From "All Good?" true path.  
  - Output: To "Extract Fields" for message preparation.  
  - Edge Cases: Large batch sizes could cause timeout or rate limits.

- **Missing/Invalid phone → Email only**  
  - Type: Split In Batches  
  - Role: Processes leads missing valid phone numbers, sending emails only.  
  - Input: From "False Email or False Phone" true path.  
  - Output: To "Extract Fields1" for email preparation.  
  - Edge Cases: Leads with missing or malformed email addresses.

- **Missing/Invalid email → SMS/WhatsApp only**  
  - Type: Split In Batches  
  - Role: Processes leads missing valid email addresses, sends SMS/WhatsApp only.  
  - Input: From "False Email or False Phone" false path.  
  - Output: To "Extract Fields2" for SMS/WhatsApp preparation.  
  - Edge Cases: Leads with missing or malformed phone numbers.

- **Sticky Note**  
  - Content: Describes three branching paths for handling different contact data completeness.

---

#### 1.4 Message Personalization & Dispatch

**Overview:**  
Personalizes lead data, sends email and/or SMS/WhatsApp messages accordingly, and updates the last run timestamp for future incremental runs.

**Nodes Involved:**  
- Extract Fields  
- Extract Fields1  
- Extract Fields2  
- Send Email  
- Send Email1  
- Send SMS/Whatsapp  
- Send SMS/Whatsapp1  
- Last Time Run (Code)  
- Sticky Note (🟧 STEP 4: Personalized Messages & Logging)

**Node Details:**

- **Extract Fields / Extract Fields1 / Extract Fields2**  
  - Type: Set  
  - Role: Extracts and assigns lead properties to standardized variables used in message bodies and recipients.  
  - Fields Extracted Include: firstName, lastName, CurrentStage, Source (with fallback), Email, Phone Number, Email Verification, Phone Verification, Task ID.  
  - Input: Batches of leads from respective Split In Batches nodes.  
  - Output: To respective messaging nodes (Send Email / Send SMS).  
  - Edge Cases: Missing or malformed fields could cause message personalization errors.

- **Send Email / Send Email1**  
  - Type: Gmail node  
  - Role: Sends personalized follow-up emails to leads’ email addresses.  
  - Configuration: Uses Gmail OAuth2 credentials; To address set from extracted Email; Subject is "Following Up!"; Message body placeholder for customization.  
  - Input: From Extract Fields nodes.  
  - Output: Some nodes route to SMS sending or last run update.  
  - Edge Cases: Gmail API quota limits, invalid email addresses, auth token expiration.

- **Send SMS/Whatsapp / Send SMS/Whatsapp1**  
  - Type: Twilio (SMS/WhatsApp)  
  - Role: Sends SMS or WhatsApp messages to lead phone numbers.  
  - Configuration: Uses Twilio credentials; From phone number preset; To phone number from extracted data; Message body placeholder; One node sends WhatsApp messages explicitly (`toWhatsapp`=true).  
  - Input: From Extract Fields nodes or Send Email node (for combined messaging).  
  - Output: Leads eventually pass to "Last Time Run" node for timestamp update.  
  - Edge Cases: Phone number format errors, Twilio API limits, messaging service errors.

- **Last Time Run (Code)**  
  - Type: Code (JavaScript)  
  - Role: Updates workflow static global data with the current timestamp marking the last successful run.  
  - Input: Triggered after message sending completes in each branch.  
  - Output: Workflow ends here for that branch.  
  - Edge Cases: Data persistence failures or simultaneous executions overwriting timestamps.

- **Sticky Note**  
  - Content: Describes sending personalized messages and logging last run timestamp for sync.

---

### 3. Summary Table

| Node Name                            | Node Type             | Functional Role                                | Input Node(s)                       | Output Node(s)                              | Sticky Note                                                                                               |
|------------------------------------|-----------------------|-----------------------------------------------|-----------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger                   | Schedule Trigger      | Triggers workflow every 3 minutes             | -                                 | Get Last time run                            | # 🟩 STEP 1: Scheduled Trigger & Lead Fetch<br>Runs on schedule and fetches latest leads from FollowUpBoss. |
| Get Last time run                 | Code                  | Retrieves last run timestamp or defaults      | Schedule Trigger                  | Get Last Lead (FUB)                          | # 🟩 STEP 1: Scheduled Trigger & Lead Fetch                                                               |
| Get Last Lead (FUB)               | HTTP Request          | Fetches new leads created after last run      | Get Last time run                 | Wait                                        | # 🟩 STEP 1: Scheduled Trigger & Lead Fetch                                                               |
| Wait                             | Wait                  | Adds delay before processing                    | Get Last Lead (FUB)               | DIvide Each Lead                             | # 🟩 STEP 1: Scheduled Trigger & Lead Fetch                                                               |
| DIvide Each Lead                 | Split Out             | Splits lead array into individual items        | Wait                             | Invalid Number and Email                     | # 🟦 STEP 2: Validation & Filtering<br>Check validity of emails and phones, filter invalid entries.         |
| Invalid Number and Email          | Filter                | Filters leads with valid email or phone        | DIvide Each Lead                 | All Good?                                    | # 🟦 STEP 2: Validation & Filtering                                                                       |
| All Good?                        | If                    | Checks if both email and phone are valid       | Invalid Number and Email          | Full data → Send both Email + SMS/WhatsApp;<br>False Email or False Phone | # 🟦 STEP 2: Validation & Filtering                                                                       |
| False Email or False Phone        | If                    | Splits leads missing phone or email             | All Good?                        | Missing/Invalid phone → Email only;<br>Missing/Invalid email → SMS/WhatsApp only | # 🟦 STEP 2: Validation & Filtering                                                                       |
| Full data → Send both Email + SMS/WhatsApp | Split In Batches      | Batches leads with both contacts valid         | All Good? (true)                 | Extract Fields                               | # 🟨 STEP 3: Smart Follow-Up Logic<br>Leads with full data get both email and SMS/WhatsApp.                  |
| Missing/Invalid phone → Email only | Split In Batches      | Batches leads missing valid phone               | False Email or False Phone (true) | Extract Fields1                              | # 🟨 STEP 3: Smart Follow-Up Logic<br>Missing phone leads get email only.                                  |
| Missing/Invalid email → SMS/WhatsApp only | Split In Batches      | Batches leads missing valid email               | False Email or False Phone (false)| Extract Fields2                              | # 🟨 STEP 3: Smart Follow-Up Logic<br>Missing email leads get SMS/WhatsApp only.                            |
| Extract Fields                   | Set                   | Extracts and formats lead fields for messaging | Full data → Send both Email + SMS/WhatsApp | Send Email                                  | # 🟧 STEP 4: Personalized Messages & Logging<br>Prepare personalized messages for leads with email & phone. |
| Extract Fields1                  | Set                   | Extracts lead fields for email only             | Missing/Invalid phone → Email only | Send Email1                                  | # 🟧 STEP 4: Personalized Messages & Logging<br>Prepare emails for leads missing phone.                    |
| Extract Fields2                  | Set                   | Extracts lead fields for SMS/WhatsApp only      | Missing/Invalid email → SMS/WhatsApp only | Send SMS/Whatsapp1                           | # 🟧 STEP 4: Personalized Messages & Logging<br>Prepare SMS/WhatsApp messages for leads missing email.     |
| Send Email                      | Gmail                  | Sends email to lead                             | Extract Fields                   | Send SMS/Whatsapp                            | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Send Email1                     | Gmail                  | Sends email to lead (email only path)           | Extract Fields1                  | Missing/Invalid phone → Email only (last run update) | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Send SMS/Whatsapp               | Twilio                 | Sends SMS/WhatsApp message                       | Send Email                      | Full data → Send both Email + SMS/WhatsApp (last run update) | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Send SMS/Whatsapp1              | Twilio                 | Sends SMS/WhatsApp message (SMS only path)      | Extract Fields2                  | Missing/Invalid email → SMS/WhatsApp only (last run update) | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Last Time Run                  | Code                   | Updates last run timestamp                       | Missing/Invalid phone → Email only;<br>Missing/Invalid email → SMS/WhatsApp only;<br>Full data → Send both Email + SMS/WhatsApp | -                                           | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Sticky Note                    | Sticky Note            | Explains branching logic                         | -                               | -                                           | # 🟨 STEP 3: Smart Follow-Up Logic<br>n8n branches into three paths based on data completeness               |
| Sticky Note1                   | Sticky Note            | Notes on personalized messaging and logging     | -                               | -                                           | # 🟧 STEP 4: Personalized Messages & Logging                                                              |
| Sticky Note2                   | Sticky Note            | Notes on validation and filtering                | -                               | -                                           | # 🟦 STEP 2: Validation & Filtering                                                                       |
| Sticky Note3                   | Sticky Note            | Notes on scheduled trigger and lead fetch        | -                               | -                                           | # 🟩 STEP 1: Scheduled Trigger & Lead Fetch                                                               |
| Sticky Note4                   | Sticky Note            | Overview of entire workflow logic and tools used | -                               | -                                           | Workflow automates lead follow-ups via FollowUpBoss, Gmail, Twilio/WhatsApp with validation and routing.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Interval: Every 3 minutes

2. **Add Code Node "Get Last time run"**  
   - Type: Code  
   - JavaScript: Retrieve last run timestamp from workflow static data or default to 15 minutes ago:  
     ```js
     const data = $getWorkflowStaticData('global');
     const lastRun = data.lastRun || new Date(Date.now() - 15 * 60 * 1000).toISOString();
     return [{ lastRun }];
     ```
   - Connect Schedule Trigger → Get Last time run

3. **Add HTTP Request Node "Get Last Lead (FUB)"**  
   - Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.followupboss.com/v1/people?createdAfter={{$json["lastRun"]}}`  
   - Authentication: HTTP Basic Auth with Follow Up Boss API credentials  
   - Connect Get Last time run → Get Last Lead (FUB)

4. **Add Wait Node**  
   - Type: Wait  
   - Duration: 6 seconds  
   - Connect Get Last Lead (FUB) → Wait

5. **Add Split Out Node "DIvide Each Lead"**  
   - Type: Split Out  
   - Field to Split Out: `people` (array from FUB response)  
   - Connect Wait → DIvide Each Lead

6. **Add Filter Node "Invalid Number and Email"**  
   - Type: Filter  
   - Condition: Pass if `emails[0].status` equals "Valid" OR `phones[0].status` equals "Valid"  
   - Connect DIvide Each Lead → Invalid Number and Email

7. **Add If Node "All Good?"**  
   - Type: If  
   - Condition:  
     - `emails[0].status` equals "Valid"  
     - AND `phones[0].status` equals "Valid"  
   - Connect Invalid Number and Email → All Good?

8. **Add If Node "False Email or False Phone"**  
   - Type: If  
   - Use to differentiate leads with missing/invalid phone vs missing/invalid email  
   - Connect All Good? → False Email or False Phone (false branch)

9. **Add Split In Batches Node "Full data → Send both Email + SMS/WhatsApp"**  
   - Type: Split In Batches  
   - Connect All Good? (true branch) → Full data → Send both Email + SMS/WhatsApp

10. **Add Split In Batches Node "Missing/Invalid phone → Email only"**  
    - Type: Split In Batches  
    - Connect False Email or False Phone (true branch) → Missing/Invalid phone → Email only

11. **Add Split In Batches Node "Missing/Invalid email → SMS/WhatsApp only"**  
    - Type: Split In Batches  
    - Connect False Email or False Phone (false branch) → Missing/Invalid email → SMS/WhatsApp only

12. **Add Set Node "Extract Fields"**  
    - For Full data leads  
    - Extract and assign: firstName, lastName, CurrentStage, Source (default to "Manually/Other" if unspecified), Email, Phone Number, Email Verification, Phone Verification, Task ID  
    - Connect Full data → Send both Email + SMS/WhatsApp → Extract Fields

13. **Add Set Node "Extract Fields1"**  
    - For Missing/Invalid phone leads  
    - Same field extraction as above  
    - Connect Missing/Invalid phone → Email only → Extract Fields1

14. **Add Set Node "Extract Fields2"**  
    - For Missing/Invalid email leads  
    - Same field extraction as above  
    - Connect Missing/Invalid email → SMS/WhatsApp only → Extract Fields2

15. **Add Gmail Node "Send Email"**  
    - Use Gmail OAuth2 credentials  
    - To: `={{ $json.Email }}`  
    - Subject: "Following Up!"  
    - Message: Customize your ideal email text  
    - Connect Extract Fields → Send Email

16. **Add Gmail Node "Send Email1"**  
    - Same configuration as Send Email  
    - Connect Extract Fields1 → Send Email1

17. **Add Twilio Node "Send SMS/Whatsapp"**  
    - Use Twilio API credentials  
    - From: e.g., "8887944798"  
    - To: `={{ $('Full data → Send both Email + SMS/WhatsApp').item.json.phones[0].value }}`  
    - Message: Customize your ideal SMS text  
    - Connect Send Email → Send SMS/Whatsapp

18. **Add Twilio Node "Send SMS/Whatsapp1"**  
    - Use Twilio API credentials  
    - From: e.g., "+18887944798"  
    - To: `=+1{{ $('Full data → Send both Email + SMS/WhatsApp').item.json.phones[0].value }}`  
    - Message: Customize your ideal SMS/WhatsApp text  
    - Enable "toWhatsapp" option (true)  
    - Connect Extract Fields2 → Send SMS/Whatsapp1

19. **Add Code Node "Last Time Run"**  
    - JavaScript: Update workflow static data with current timestamp  
    ```js
    const data = $getWorkflowStaticData('global');
    data.lastRun = new Date().toISOString();
    return items;
    ```  
    - Connect Send SMS/Whatsapp, Send Email1, and Send SMS/Whatsapp1 nodes to Last Time Run node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates your entire lead follow-up process across email, SMS, and WhatsApp. It pulls latest leads from FollowUpBoss, validates contact data, and sends personalized messages via Gmail and Twilio. Scales from 10 to 10,000+ leads effortlessly. Replace API keys before use. | Workflow Overview Sticky Note                                                                                                    |
| Step 1: Scheduled Trigger & Lead Fetch – Runs every 3 minutes, fetching new leads since last run.                                                                                                                                                                                     | Sticky Note near Schedule Trigger                                                                                               |
| Step 2: Validation & Filtering – Filters leads with invalid or missing emails/phones to avoid wasted messages and bounces.                                                                                                                                                             | Sticky Note near validation nodes                                                                                               |
| Step 3: Smart Follow-Up Logic – Routes leads into three paths for combined email+SMS, email only, or SMS/WhatsApp only messaging.                                                                                                                                                     | Sticky Note near branching nodes                                                                                                |
| Step 4: Personalized Messages & Logging – Sends customized messages and updates last run timestamp to maintain sync.                                                                                                                                                                    | Sticky Note near messaging nodes                                                                                                |
| Tools used: FollowUpBoss API, Gmail OAuth2, Twilio API for SMS/WhatsApp, n8n automation platform.                                                                                                                                                                                      | Workflow Overview Sticky Note                                                                                                    |
| Gmail OAuth2 and Twilio credentials must be valid and properly configured to send messages. Phone numbers should be in E.164 format for Twilio.                                                                                                                                         | Credential Notes                                                                                                                  |
| Consider API rate limits and handle possible errors like auth failures or invalid data to ensure robustness.                                                                                                                                                                           | General robustness advice                                                                                                        |

---

This documentation provides a detailed reference to understand, reproduce, and modify the automated lead follow-up workflow in n8n. It clearly explains the functional blocks, node roles, configurations, and potential issues for robust operation.