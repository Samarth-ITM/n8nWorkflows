Pre-Meeting Email Nurturing Sequence with Cal.com and Gmail

https://n8nworkflows.xyz/workflows/pre-meeting-email-nurturing-sequence-with-cal-com-and-gmail-10795


# Pre-Meeting Email Nurturing Sequence with Cal.com and Gmail

### 1. Workflow Overview

This workflow automates a **Pre-Meeting Email Nurturing Sequence** triggered by new bookings in Cal.com. It is designed to engage and warm up prospects from the moment they schedule a meeting until the meeting day, improving attendance and setting expectations.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Prospect Extraction:** Captures booking data from Cal.com via webhook, extracts attendee and meeting details, and identifies if the email is a company domain.
  
- **1.2 Time Calculation and Data Aggregation:** Calculates how many days remain until the meeting and compiles prospect and AI-derived data for downstream logic.
  
- **1.3 Conditional Routing by Days Until Meeting:** Uses a Switch node to route the flow into different email sequences based on how many days remain before the meeting (0–1 day, 2–7 days, 8+ days).
  
- **1.4 Email Sequence Blocks:** Each time-range branch sends a series of Gmail emails spaced by Wait nodes with customized nurturing messages tailored to the timing and prospect context.

- **1.5 Wait & Delay Handling:** Uses Wait nodes to space out emails naturally, preventing message overload and simulating a drip campaign.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Prospect Extraction

**Overview:**  
This block triggers on new Cal.com bookings, extracts attendee and meeting information, checks if the attendee’s email is a company email or consumer provider, and constructs a website URL guess.

**Nodes Involved:**  
- Cal.com Trigger  
- Extract prospect  
- If (checks company email)

**Node Details:**

- **Cal.com Trigger**  
  - Type: Cal.com webhook trigger  
  - Role: Listens for `BOOKING_CREATED` events from Cal.com to start the workflow.  
  - Config: Uses stored Cal.com API credentials.  
  - Input: Webhook payload from Cal.com booking.  
  - Output: Raw booking JSON data.  
  - Failures: Webhook misconfiguration, Cal.com API downtime or auth issues.  
  - Version: 2  

- **Extract prospect**  
  - Type: Code (JavaScript)  
  - Role: Parses booking JSON to extract attendee email, name, meeting info, detects company email vs consumer email domains, and guesses website URL based on email or company name field.  
  - Key expressions:  
    - Extracts email domain from attendee email.  
    - Checks domain against list of common consumer providers (gmail, yahoo, outlook, etc.).  
    - Constructs website URL heuristically.  
  - Input: Cal.com Trigger output.  
  - Output: JSON with attendee details, company name, email domain, boolean `is_company_email`, and `website_url`.  
  - Failures: Unexpected JSON structure, missing fields, or malformed emails.  
  - Version: 2  

- **If**  
  - Type: Conditional logic node  
  - Role: Checks if `is_company_email` is true to continue processing only company emails.  
  - Config: Condition `{{$json.is_company_email}}` is true.  
  - Input: Extract prospect output.  
  - Output: Passes through only company emails for further processing.  
  - Failures: Expression errors if `is_company_email` is undefined.  
  - Version: 2.2  

---

#### 2.2 Time Calculation and Data Aggregation

**Overview:**  
Calculates days remaining until the meeting and merges prospect data with AI-sourced analysis for personalization.

**Nodes Involved:**  
- Calculate time

**Node Details:**

- **Calculate time**  
  - Type: Code (JavaScript)  
  - Role: Reads meeting start time, computes days until meeting, merges with AI data (presumably from another input or previous node), outputs enriched JSON for email content.  
  - Key expressions:  
    - Calculates difference between meeting date and current date in days.  
    - Preserves key fields like attendee email, company name, website URL, and AI-derived insights (business type, tech stack, pain points, personalization hooks).  
  - Input: Output from `Extract prospect` and AI data input.  
  - Output: JSON enriched with `days_until_meeting` and other details.  
  - Failures: Date parsing errors, missing AI data, time zone issues.  
  - Version: 2  

---

#### 2.3 Conditional Routing by Days Until Meeting

**Overview:**  
Routes the workflow to different email sequences based on the number of days until the meeting.

**Nodes Involved:**  
- Switch: Days Until Meeting

**Node Details:**

- **Switch: Days Until Meeting**  
  - Type: Switch (Conditional routing)  
  - Role: Routes flow into three branches:  
    - "0–1 days" (imminent meetings)  
    - "2–7 days" (mid-range)  
    - "8+ days" (long lead time)  
  - Config:  
    - <= 1 day: route 0-1 days  
    - >1 and <=7 days: route 2-7 days  
    - >7 days: route 8+ days  
  - Input: Output from `Calculate time`.  
  - Output: Routes to respective email sequences.  
  - Failures: Expression evaluation errors, numeric type issues.  
  - Version: 3  

---

#### 2.4 Email Sequence Blocks

**Overview:**  
Each branch sends a series of pre-configured nurturing emails via Gmail spaced by Wait nodes. Email content is personalized using attendee and meeting data.

**Nodes Involved:**  
- Quick prep email (Set node)  
- Send it (Gmail)  
- Casual flex (Gmail)  
- Casual press (Gmail)  
- Casual knowledge flex (Gmail)  
- Wait nodes used to space emails  
- Similar sets and Gmail nodes for 8+ days branch (e.g., Quick prep email2, Casual flex 8+ days, Casual press 8+ days, etc.)

**Node Details:**

- **Quick prep email / Quick prep email (2) / Quick prep email2**  
  - Type: Set  
  - Role: Builds a personalized email message string including recipient first name, company, meeting time in friendly format.  
  - Key expressions:  
    - Extracts first name from full name.  
    - Formats meeting time as "X time tomorrow" or "X time on Weekday".  
    - Inserts personalization questions to increase engagement.  
  - Input: Data from Calculate time or Wait nodes.  
  - Output: JSON with `email` string field for sending.  
  - Failures: String manipulation errors, missing fields.  
  - Version: 3.4  

- **Send it / Quick prep / Quick prep +8days**  
  - Type: Gmail  
  - Role: Sends the prepared email to attendee’s email via Gmail OAuth2.  
  - Config:  
    - Recipient: attendee email from JSON.  
    - Subject: e.g., "Quick prep for tomorrow's call 🙏🏻"  
    - Message: from Set node output.  
    - Email type: text/plain, no HTML.  
  - Credentials: Google Gmail OAuth2 credentials required.  
  - Failures: Auth errors, API limits, invalid recipient emails, network issues.  
  - Version: 2.1  

- **Casual flex / Casual flex 8+ days**  
  - Type: Gmail  
  - Role: Sends a casual, informal email encouraging reflection on ideal client profile and prep for call.  
  - Content: Pre-written text with some personalization variables (company name).  
  - Input: Wait node outputs.  
  - Failures: Same as Gmail nodes above.  
  - Version: 2.1  

- **Casual press / Casual press 8+days**  
  - Type: Gmail  
  - Role: Sends an introductory, context-setting email about the sender's background and approach.  
  - Content: Pre-written, personalized with recipient first name.  
  - Failures: Same as Gmail nodes above.  
  - Version: 2.1  

- **Casual knowledge flex / Casual knowledge flex 8+days**  
  - Type: Gmail  
  - Role: Sends an insightful email addressing common pain points and offering tactics or frameworks.  
  - Content: Includes dynamic insertion of business type and likely pain points from AI data.  
  - Failures: Missing AI data could lead to incomplete messages.  
  - Version: 2.1  

---

#### 2.5 Wait & Delay Handling

**Overview:**  
Wait nodes space out emails to create a natural drip campaign over days.

**Nodes Involved:**  
- Wait 1 Day  
- Wait another day  
- Wait 1 Day1  
- Wait another day 8+days  
- Wait (generic)  
- Wait1  

**Node Details:**

- **Wait nodes**  
  - Type: Wait  
  - Role: Pause the workflow execution for a fixed duration (mostly 1 day).  
  - Config: Default 1-day wait before triggering next email.  
  - Input/Output: Connects email sends in sequence.  
  - Failures: Workflow timeouts or server restarts can interrupt waits.  
  - Version: 1.1  

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                                   | Input Node(s)                | Output Node(s)                      | Sticky Note                                                                                                                        |
|-----------------------------|-----------------------|-------------------------------------------------|------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Cal.com Trigger             | Cal.com Trigger       | Trigger workflow on new Cal.com booking          | (Webhook)                    | Extract prospect                   | # ⭐ Overview Automated Pre-Meeting Email Sequence for Cal.com Bookings... (full content in Sticky Note node)                      |
| Extract prospect            | Code                  | Extract attendee and meeting info; detect email type | Cal.com Trigger             | If                                |                                                                                                                                    |
| If                         | If                    | Check if attendee email is company email         | Extract prospect             | Calculate time                    |                                                                                                                                    |
| Calculate time             | Code                  | Calculate days until meeting; merge AI data      | If                          | Switch: Days Until Meeting        |                                                                                                                                    |
| Switch: Days Until Meeting | Switch                | Route based on days until meeting                 | Calculate time               | Quick prep email, Casual flex, Casual flex 8+ days |                                                                                                                                    |
| Quick prep email           | Set                   | Compose personalized quick prep email body       | Switch                      | Send it                          | ## Nurturing Message You can change the message for your needs and duration before sending the messages in here.                  |
| Send it                   | Gmail                  | Send quick prep email                             | Quick prep email             | Wait 1 Day                      |                                                                                                                                    |
| Wait 1 Day                | Wait                   | Wait 1 day before next email                      | Send it                     | Casual press                    |                                                                                                                                    |
| Casual press              | Gmail                  | Send casual introduction email                    | Wait 1 Day                  | Wait another day                |                                                                                                                                    |
| Wait another day          | Wait                   | Wait 1 day                                        | Casual press                | Casual knowledge flex           |                                                                                                                                    |
| Casual knowledge flex     | Gmail                  | Send knowledge sharing email                       | Wait another day            | Wait                          |                                                                                                                                    |
| Wait                      | Wait                   | Wait 1 day                                        | Casual knowledge flex        | Quick prep email (2)             |                                                                                                                                    |
| Quick prep email (2)       | Set                   | Compose 2nd quick prep email                       | Wait                        | Quick prep                      |                                                                                                                                    |
| Quick prep                 | Gmail                  | Send 2nd quick prep email                          | Quick prep email (2)         | Wait                          |                                                                                                                                    |
| Casual flex               | Gmail                  | Send casual flex email for 2–7 days branch       | Switch                      | Wait 1 Day                     |                                                                                                                                    |
| Wait 1 Day1               | Wait                   | Wait 1 day for 8+ days branch                     | Casual flex 8+ days          | Casual press 8+days             |                                                                                                                                    |
| Casual flex 8+ days        | Gmail                  | Send casual flex email for 8+ days branch         | Switch                      | Wait 1 Day1                    |                                                                                                                                    |
| Casual press 8+days        | Gmail                  | Send casual press email for 8+ days branch        | Wait 1 Day1                 | Wait another day  8+days        |                                                                                                                                    |
| Wait another day  8+days   | Wait                   | Wait 1 day                                        | Casual press 8+days          | Casual knowledge flex 8+days    |                                                                                                                                    |
| Casual knowledge flex 8+days| Gmail                 | Send knowledge flex email for 8+ days branch      | Wait another day  8+days     | Wait1                         |                                                                                                                                    |
| Wait1                     | Wait                   | Wait 1 day                                        | Casual knowledge flex 8+days | Quick prep email2              |                                                                                                                                    |
| Quick prep email2          | Set                   | Compose quick prep email for 8+ days branch       | Wait1                       | Quick prep +8days              |                                                                                                                                    |
| Quick prep +8days          | Gmail                  | Send quick prep email for 8+ days branch           | Quick prep email2            |                               |                                                                                                                                    |
| Sticky Note               | Sticky Note            | Overview and detailed workflow explanation        |                              |                               | # ⭐ Overview Automated Pre-Meeting Email Sequence for Cal.com Bookings...                                                         |
| Sticky Note1              | Sticky Note            | Note about customizable nurturing message          |                              |                               | ## Nurturing Message You can change the message for your needs and duration before sending the messages in here.                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cal.com Trigger Node**  
   - Type: Cal.com Trigger  
   - Configure to listen for `BOOKING_CREATED` event.  
   - Add your Cal.com API credentials (OAuth or API key).  
   - Position as the workflow start trigger.

2. **Add Code Node “Extract prospect”**  
   - Input: Cal.com Trigger output.  
   - JavaScript: Parse webhook payload to extract attendee email, name, meeting start time, title, company name from custom fields, and determine if email is company or consumer provider.  
   - Extract domain from email and create a website URL guess.  
   - Output the fields: attendee_email, attendee_name, meeting_start_time, meeting_title, email_domain, company_name, is_company_email, website_url.

3. **Add If Node “If”**  
   - Input: Extract prospect output.  
   - Condition: `is_company_email` is true (boolean).  
   - Pass only company emails to next step.

4. **Add Code Node “Calculate time”**  
   - Input: If node output.  
   - JavaScript: Calculate days until meeting by subtracting today's date from meeting_start_time.  
   - Merge prospect data and AI data (if available) into output JSON including fields for personalization like pain points, business type, and tech stack.  
   - Output includes days_until_meeting field.

5. **Add Switch Node “Switch: Days Until Meeting”**  
   - Input: Calculate time output.  
   - Setup three outputs:  
     - 0–1 days: days_until_meeting <= 1  
     - 2–7 days: 1 < days_until_meeting <= 7  
     - 8+ days: days_until_meeting > 7  
   - Route flow according to these conditions.

6. **For 0–1 days branch:**  
   a. Add Set node “Quick prep email”: Compose personalized email with first name, company, meeting time formatted as “time tomorrow” or “time on weekday”. Include preparatory questions.  
   b. Add Gmail node “Send it”: Send email with subject “Quick prep for tomorrow's call 🙏🏻” using Gmail OAuth2 credentials.  
   c. Add Wait node “Wait 1 Day”: Pause 1 day.  
   d. Add Gmail node “Casual press”: Send casual intro email.  
   e. Add Wait node “Wait another day”: Pause 1 day.  
   f. Add Gmail node “Casual knowledge flex”: Send knowledge sharing email with pain points, business type.  
   g. Add Wait node “Wait”: Pause 1 day.  
   h. Add Set node “Quick prep email (2)”: Compose second quick prep email.  
   i. Add Gmail node “Quick prep”: Send second quick prep email.

7. **For 2–7 days branch:**  
   a. Add Gmail node “Casual flex”: Send casual flex email encouraging thinking about ideal client profile.  
   b. Connect to Wait node “Wait 1 Day”.  
   c. Connect to Gmail node “Casual press”.  
   d. Continue as per 0–1 days branch or adjust similarly as needed.

8. **For 8+ days branch:**  
   a. Add Gmail node “Casual flex 8+ days”: Send light casual flex email.  
   b. Add Wait node “Wait 1 Day1”.  
   c. Add Gmail node “Casual press 8+days”: Send casual press email.  
   d. Add Wait node “Wait another day 8+days”.  
   e. Add Gmail node “Casual knowledge flex 8+days”: Send knowledge email with dynamic pain points and business type.  
   f. Add Wait node “Wait1”.  
   g. Add Set node “Quick prep email2”: Compose quick prep email for 8+ days.  
   h. Add Gmail node “Quick prep +8days”: Send quick prep email.

9. **Connect all nodes appropriately according to the workflow diagram.**

10. **Add Sticky Note nodes for documentation and guidance if desired.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| # ⭐ Overview Automated Pre-Meeting Email Sequence for Cal.com Bookings: This workflow automatically warms leads by sending timed, personalized emails after booking. Setup requires Cal.com API key and Gmail OAuth2 credentials. Customize messages freely.                                                                                                                                                                         | Sticky Note at workflow start                                                                    |
| ## Nurturing Message: You can change the message for your needs and duration before sending the messages here.                                                                                                                                                                                                                                                                                                                    | Sticky Note near email Set nodes                                                                 |
| Gmail nodes require OAuth2 credentials setup in n8n with appropriate scopes to send emails on your behalf. Ensure credentials are kept secure.                                                                                                                                                                                                                                                                                    | General credential note                                                                           |
| Wait nodes may cause workflow to pause for extended periods. Ensure your n8n instance supports long-running workflows and persistence to avoid dropped executions.                                                                                                                                                                                                                                                               | General workflow reliability note                                                                |
| Email personalization relies on AI data integration for pain points and business type. You may need to integrate external AI workflows or nodes to supply this data before `Calculate time`.                                                                                                                                                                                                                                       | Integration note                                                                                  |
| Cal.com webhook requires proper configuration in Cal.com dashboard to send booking events to your n8n webhook URL.                                                                                                                                                                                                                                                                                                                | Integration setup note                                                                           |

---

**Disclaimer:** The provided content is derived solely from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal or protected material. All handled data is legal and publicly accessible.