Hotel Guest Journey Automation with Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/hotel-guest-journey-automation-with-gmail---google-sheets-6631


# Hotel Guest Journey Automation with Gmail & Google Sheets

### 1. Workflow Overview

This workflow automates the hotel guest communication journey using Gmail and Google Sheets data. It targets hotel managers and hospitality businesses aiming to enhance guest experience, reduce manual work, and improve operational efficiency through timely, personalized emails and daily staff reports.

The workflow is logically divided into three main blocks:

- **1.1 Pre-Arrival Welcome Email Automation**  
  Automatically sends personalized welcome emails 1-2 days prior to guest check-in, including reservation details and hotel amenities.

- **1.2 Post-Stay Review Request Automation**  
  Sends review request emails 24 hours after guest check-out, encouraging online reviews and offering return guest discounts.

- **1.3 Daily Arrivals & Departures Report Automation**  
  Generates and emails a comprehensive daily report every morning at 6:00 AM for front desk, housekeeping, and management staff.

Each block monitors reservation data in a Google Sheet, applies filtering logic based on dates and email sent status, sends emails via Gmail, and updates the Google Sheet to track sent communications, preventing duplicates.

---

### 2. Block-by-Block Analysis

#### 2.1 Pre-Arrival Welcome Email Automation

- **Overview:**  
  This block triggers every 6 hours to check for guests whose check-in date is within the next 1-2 days and who have not yet received a welcome email. It sends a personalized welcome email with reservation details and hotel amenities, then marks the email as sent in the Google Sheet.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Edit Fields  
  - Get Hotel Reservation (Google Sheets)  
  - Filter Upcoming Guest  
  - Switch  
  - Filter Unsent Welcome Emails  
  - Welcome Email (Gmail)  
  - Mark Welcome Email as Sent (Google Sheets)  
  - Sticky Note (overview)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 6 hours  
    - Configuration: Interval set to 6 hours  
    - Inputs: None  
    - Outputs: Triggers Edit Fields node  
    - Failures: Cron misconfiguration unlikely; runtime errors possible if n8n environment unavailable

  - **Edit Fields**  
    - Type: Set  
    - Role: Adds hotel name field ("Mr. Wonderful") to data  
    - Configuration: Sets "Hotel Name" = "Mr. Wonderful"; retains other fields  
    - Input: Trigger from Schedule Trigger  
    - Output: Data enriched with hotel name to Get Hotel Reservation  
    - Failures: Expression errors unlikely

  - **Get Hotel Reservation**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves reservation rows from Google Sheet "Hotel Bookings" (sheet ID and gid=0)  
    - Configuration: Filters for rows where either "Welcome Email Sent" or "Write Review Email Sent" is empty (OR condition)  
    - Input: Data with hotel name  
    - Output: List of reservations needing email action  
    - Failures: Google Sheets API auth errors, quota limits, or missing sheet

  - **Filter Upcoming Guest**  
    - Type: Filter  
    - Role: Filters reservations where either "Welcome Email Sent" or "Write Review Email Sent" is empty (OR), passing on rows needing email  
    - Input: Reservations list  
    - Output: Filtered list to Switch node  
    - Failures: Filter expression errors if data missing

  - **Switch**  
    - Type: Switch  
    - Role: Routes records either to "welcome email" or "Write Review" branch based on date logic:  
      - "welcome email": Check-in is >0 and ≤2 days from now  
      - "Write Review": Check-out is ≥1 and ≤2 days ago  
    - Expressions: Uses DateTime to parse dates from format 'M/d/yyyy HH:mm:ss' and compares with current date  
    - Input: Filtered upcoming guests  
    - Output: Two branches  
    - Failures: Date parsing errors, empty or malformed date fields

  - **Filter Unsent Welcome Emails**  
    - Type: Filter  
    - Role: Allows only reservations where "Welcome Email Sent" is empty  
    - Input: Switch output "welcome email" branch  
    - Output: Eligible guests for welcome email  
    - Failures: Expression failure if field missing

  - **Welcome Email (Gmail)**  
    - Type: Gmail  
    - Role: Sends personalized HTML welcome email to guest's email address  
    - Configuration:  
      - Recipient: `Guest Email` field  
      - Subject: "Welcome to Hotel - Your Stay Begins Soon! 🏨"  
      - Message: Fully styled HTML email including guest name, booking details, amenities, and contact info. Hotel name placeholder replaced by workflow field.  
    - Credentials: Gmail OAuth2 account configured  
    - Input: Filtered guests missing welcome email  
    - Output: Passes data to Mark Welcome Email as Sent  
    - Failures: Gmail API auth errors, send quota reached, invalid email format

  - **Mark Welcome Email as Sent (Google Sheets)**  
    - Type: Google Sheets (Update)  
    - Role: Updates the reservation row to mark "Welcome Email Sent" as "yes" and records timestamp  
    - Configuration: Matches by "Booking ID"; writes to columns "Welcome Email Sent" and "Welcome Email Date" with current date/time in Asia/Kolkata timezone  
    - Input: From Welcome Email node  
    - Output: End of welcome email branch  
    - Failures: Update conflicts, Google Sheets API errors

  - **Sticky Note**  
    - Provides workflow documentation on the pre-arrival welcome email automation purpose and benefits.

---

#### 2.2 Post-Stay Review Request Automation

- **Overview:**  
  This block sends review request emails automatically 24 hours after guest check-out for guests who haven't yet received a review request email. It includes Google Reviews and TripAdvisor links, discount offers, and contact info, then updates the sheet to avoid duplicates.

- **Nodes Involved:**  
  - Switch (same as above)  
  - Filter Unsent Review Requests  
  - Write Review (Gmail)  
  - Mark Review Email as Sent (Google Sheets)  
  - Sticky Note (overview)

- **Node Details:**

  - **Switch**  
    - As described above, routes guests to the "Write Review" branch when check-out date is 1-2 days ago.

  - **Filter Unsent Review Requests**  
    - Type: Filter  
    - Role: Filters guests who have not yet received a review request email  
    - Condition: "Write Review Email Sent" field is empty  
    - Input: Switch "Write Review" output  
    - Output: Eligible guests for review request email  
    - Failures: Expression errors if field missing

  - **Write Review (Gmail)**  
    - Type: Gmail  
    - Role: Sends a detailed review request email with personalized content and booking info  
    - Configuration:  
      - Recipient: Guest Email  
      - Subject: Personalized with guest name and hotel name  
      - Message: Styled HTML encouraging reviews, offering discounts, and including contact details  
    - Credentials: Gmail OAuth2  
    - Input: Filtered guests needing review email  
    - Output: To Mark Review Email as Sent node  
    - Failures: Gmail API issues, invalid email, quota

  - **Mark Review Email as Sent (Google Sheets)**  
    - Type: Google Sheets (Update)  
    - Role: Updates the sheet marking "Write Review Email Sent" as "yes" and records timestamp  
    - Configuration: Matched by Booking ID; writes "Write Review Email Sent" and date/time  
    - Input: From Write Review node  
    - Output: End of review email branch  
    - Failures: Google Sheets API errors

  - **Sticky Note1**  
    - Describes the review request automation goals and how it benefits hotel reputation and guest loyalty.

---

#### 2.3 Daily Arrivals & Departures Report Automation

- **Overview:**  
  Runs every day at 6 AM to generate a detailed report summarizing today's arrivals and departures, housekeeping priorities, and front desk notes. The report is emailed to staff addresses for operational awareness.

- **Nodes Involved:**  
  - Daily at 6 AM (Schedule Trigger)  
  - Add Information (Set)  
  - Get All Reservations (Google Sheets)  
  - Filter Today's Arrivals  
  - Filter Today's Departures  
  - Create Email HTML (Set)  
  - Send Daily Staff Report (Gmail)  
  - Sticky Note2 (overview)  
  - Sticky Note3 (overall workflow description)

- **Node Details:**

  - **Daily at 6 AM (Schedule Trigger)**  
    - Type: Schedule Trigger  
    - Role: Starts the daily report workflow every day at 6:00 AM  
    - Configuration: Cron expression "0 6 * * *"  
    - Input: None  
    - Output: Triggers Add Information  
    - Failures: Cron misconfiguration or environment downtime

  - **Add Information (Set)**  
    - Type: Set  
    - Role: Adds static information such as hotel name ("Mr. Wonderful") and staff email ("staff@gmail.com") for report distribution  
    - Configuration: Sets "Hotel Name" and "staff mail a" fields  
    - Input: From daily trigger  
    - Output: To Get All Reservations  
    - Failures: None expected

  - **Get All Reservations (Google Sheets)**  
    - Type: Google Sheets (Read)  
    - Role: Fetches all reservation data from the Google Sheet for processing  
    - Configuration: Reads all rows from "Reservations" sheet (gid=0)  
    - Input: From Add Information  
    - Output: To filters for arrivals and departures  
    - Failures: API errors, auth

  - **Filter Today's Arrivals**  
    - Type: Filter  
    - Role: Filters reservations with check-in date matching today  
    - Configuration: Compares check-in date formatted as "M/d/yyyy" with today's date  
    - Input: Reservation list  
    - Output: Filtered arrivals to Create Email HTML  
    - Failures: Date parsing errors

  - **Filter Today's Departures**  
    - Type: Filter  
    - Role: Filters reservations with check-out date matching today  
    - Configuration: Same date format filtering as arrivals  
    - Input: Reservation list  
    - Output: Filtered departures to Create Email HTML  
    - Failures: Date parsing errors

  - **Create Email HTML (Set)**  
    - Type: Set  
    - Role: Constructs the full HTML body for the daily operations report email  
    - Configuration: Uses data from filtered arrivals and departures to summarize:  
      - Number of arrivals and departures today  
      - Guest details for arrivals/departures  
      - Housekeeping priorities (e.g., rooms to clean after departures)  
      - Front desk notes (totals, peak times)  
      - Includes styling and branding with hotel name  
    - Input: From both Filter Today's Arrivals and Filter Today's Departures nodes  
    - Output: To Send Daily Staff Report  
    - Failures: Template or expression errors in HTML composition

  - **Send Daily Staff Report (Gmail)**  
    - Type: Gmail  
    - Role: Sends the generated daily report email to the staff email  
    - Configuration:  
      - Recipient: Configured staff mail from Add Information node  
      - Subject: Dynamic, includes day of month and month from Add Information  
      - Message: The HTML daily report content from Create Email HTML  
    - Credentials: Gmail OAuth2  
    - Input: Email HTML content  
    - Output: Workflow end for daily report branch  
    - Failures: Gmail API errors, invalid email

  - **Sticky Note2**  
    - Describes the daily arrivals/departures report automation purpose and benefits.

  - **Sticky Note3**  
    - Provides an overall workflow description, setup instructions, target users, and customization guidance.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                             | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                         |
|---------------------------|---------------------------|---------------------------------------------|-----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Sticky Note               | Sticky Note               | Overview of Pre-Arrival Welcome Email Block | None                        | None                            | 🏨 PRE-ARRIVAL WELCOME EMAIL AUTOMATION - Personalized welcome emails before check-in, updates every 6 hours       |
| Schedule Trigger          | Schedule Trigger          | Triggers pre-arrival email check every 6h  | None                        | Edit Fields                     |                                                                                                                    |
| Edit Fields              | Set                       | Adds hotel name field                        | Schedule Trigger            | Get Hotel Reservation           |                                                                                                                    |
| Get Hotel Reservation     | Google Sheets             | Reads reservations needing email             | Edit Fields                 | Filter Upcoming Guest           |                                                                                                                    |
| Filter Upcoming Guest     | Filter                    | Filters reservations missing welcome or review emails | Get Hotel Reservation       | Switch                         |                                                                                                                    |
| Switch                   | Switch                    | Routes records to welcome email or review request branch | Filter Upcoming Guest       | Filter Unsent Welcome Emails, Filter Unsent Review Requests |                                                                                                                    |
| Filter Unsent Welcome Emails | Filter                  | Filters for reservations without welcome email sent | Switch (welcome email)      | Welcome Email                  |                                                                                                                    |
| Welcome Email            | Gmail                     | Sends personalized welcome email             | Filter Unsent Welcome Emails | Mark Welcome Email as Sent      |                                                                                                                    |
| Mark Welcome Email as Sent | Google Sheets             | Marks welcome email as sent in sheet         | Welcome Email               | None                           |                                                                                                                    |
| Filter Unsent Review Requests | Filter                  | Filters for reservations without review email sent | Switch (Write Review)       | Write Review                   |                                                                                                                    |
| Write Review             | Gmail                     | Sends post-stay review request email         | Filter Unsent Review Requests | Mark Review Email as Sent       |                                                                                                                    |
| Mark Review Email as Sent | Google Sheets             | Marks review email as sent in sheet           | Write Review                | None                           |                                                                                                                    |
| Sticky Note1             | Sticky Note               | Overview of Post-Stay Review Request Automation | None                        | None                           | ⭐ POST-STAY REVIEW REQUEST AUTOMATION - Boosts online reputation and guest loyalty                                 |
| Daily at 6 AM            | Schedule Trigger          | Triggers daily report generation at 6 AM     | None                        | Add Information                |                                                                                                                    |
| Add Information          | Set                       | Adds static info like hotel name, staff email | Daily at 6 AM              | Get All Reservations           |                                                                                                                    |
| Get All Reservations     | Google Sheets             | Reads all reservations for daily report       | Add Information             | Filter Today's Arrivals, Filter Today's Departures |                                                                                                                    |
| Filter Today's Arrivals  | Filter                    | Filters reservations with check-in today      | Get All Reservations        | Create Email HTML              |                                                                                                                    |
| Filter Today's Departures | Filter                    | Filters reservations with check-out today     | Get All Reservations        | Create Email HTML              |                                                                                                                    |
| Create Email HTML        | Set                       | Builds HTML daily report email content         | Filter Today's Arrivals, Filter Today's Departures | Send Daily Staff Report       |                                                                                                                    |
| Send Daily Staff Report  | Gmail                     | Sends daily report email to staff              | Create Email HTML           | None                           |                                                                                                                    |
| Sticky Note2             | Sticky Note               | Overview of Daily Arrivals & Departures Report | None                        | None                           | 📋 DAILY ARRIVALS & DEPARTURES AUTOMATION - Morning report for staff with arrivals, departures, housekeeping info |
| Sticky Note3             | Sticky Note               | Overall workflow description and setup guide  | None                        | None                           | 🏨 HOTEL GUEST COMMUNICATION AUTOMATION - Complete guest journey, setup, and customization notes                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow** named "Hotel Booking System" and set timezone to "Asia/Kolkata".

2. **Create Sticky Note (Overview #3):** Add overall workflow description with setup instructions and target users.

3. **Pre-Arrival Welcome Email Block:**

   a. Add **Schedule Trigger** node:  
      - Type: Schedule Trigger  
      - Trigger every 6 hours (set interval field: hoursInterval = 6)  

   b. Add **Set (Edit Fields)** node:  
      - Add field "Hotel Name" with value "Mr. Wonderful"  
      - Include other existing fields  

   c. Connect Schedule Trigger → Edit Fields.

   d. Add **Google Sheets (Get Hotel Reservation)** node:  
      - Credentials: Google Sheets OAuth2 account  
      - Document ID: Your Google Sheet ID containing reservations  
      - Sheet Name: "Reservations" or appropriate sheet (gid=0)  
      - Operation: Read rows  
      - Filter: Rows where "Welcome Email Sent" OR "Write Review Email Sent" is empty (OR condition on these columns)  

   e. Connect Edit Fields → Get Hotel Reservation.

   f. Add **Filter (Filter Upcoming Guest)** node:  
      - Conditions: "Welcome Email Sent" is empty OR "Write Review Email Sent" is empty (OR)  

   g. Connect Get Hotel Reservation → Filter Upcoming Guest.

   h. Add **Switch** node:  
      - Add two outputs with rules:  
        1. "welcome email": Check if days until check-in date is >0 and ≤2  
        2. "Write Review": Check if days since check-out date is ≥1 and ≤2  
      - Use DateTime expressions parsing "M/d/yyyy HH:mm:ss" format for dates  

   i. Connect Filter Upcoming Guest → Switch.

   j. Add **Filter (Filter Unsent Welcome Emails)** node:  
      - Condition: "Welcome Email Sent" is empty  

   k. Connect Switch "welcome email" output → Filter Unsent Welcome Emails.

   l. Add **Gmail (Welcome Email)** node:  
      - Credentials: Gmail OAuth2  
      - To: Guest Email from JSON data  
      - Subject: "Welcome to Hotel - Your Stay Begins Soon! 🏨"  
      - Message: Full HTML email with placeholders for guest name, booking details, amenities, and contact info  
      - Replace "[Hotel Name]" placeholders with the "Hotel Name" field  

   m. Connect Filter Unsent Welcome Emails → Welcome Email.

   n. Add **Google Sheets (Mark Welcome Email as Sent)** node:  
      - Credentials: Google Sheets OAuth2  
      - Operation: Update row  
      - Matching columns: Booking ID  
      - Columns to update: "Welcome Email Sent" = "yes", "Welcome Email Date" = current datetime (Asia/Kolkata timezone in format "M/d/yy HH:mm:ss")  

   o. Connect Welcome Email → Mark Welcome Email as Sent.

4. **Post-Stay Review Request Block:**

   a. Add **Filter (Filter Unsent Review Requests)** node:  
      - Condition: "Write Review Email Sent" is empty  

   b. Connect Switch "Write Review" output → Filter Unsent Review Requests.

   c. Add **Gmail (Write Review)** node:  
      - Credentials: Gmail OAuth2  
      - To: Guest Email  
      - Subject: Personalized with guest name and hotel name  
      - Message: HTML email requesting reviews with booking info and discount code  

   d. Connect Filter Unsent Review Requests → Write Review.

   e. Add **Google Sheets (Mark Review Email as Sent)** node:  
      - Credentials: Google Sheets OAuth2  
      - Operation: Update row  
      - Matching columns: Booking ID  
      - Columns to update: "Write Review Email Sent" = "yes", "Write Review Email Date" = current datetime (Asia/Kolkata timezone)  

   f. Connect Write Review → Mark Review Email as Sent.

5. **Daily Arrivals & Departures Report Block:**

   a. Add **Schedule Trigger (Daily at 6 AM)** node:  
      - Cron expression: "0 6 * * *"  

   b. Add **Set (Add Information)** node:  
      - Add fields:  
        - "Hotel Name" = "Mr. Wonderful"  
        - "staff mail a" = staff email address (e.g., "staff@gmail.com")  
      - Include other fields as needed  

   c. Connect Daily at 6 AM → Add Information.

   d. Add **Google Sheets (Get All Reservations)** node:  
      - Credentials: Google Sheets OAuth2  
      - Read all rows from the reservation sheet  

   e. Connect Add Information → Get All Reservations.

   f. Add **Filter (Filter Today's Arrivals)** node:  
      - Condition: Check-in date (formatted "M/d/yyyy") equals current date (formatted "M/d/yyyy")  

   g. Add **Filter (Filter Today's Departures)** node:  
      - Condition: Check-out date equals current date  

   h. Connect Get All Reservations → Filter Today's Arrivals  
   i. Connect Get All Reservations → Filter Today's Departures

   j. Add **Set (Create Email HTML)** node:  
      - Compose HTML email body that:  
        - Summarizes number of arrivals and departures today  
        - Lists guest details for arrivals and departures  
        - Includes housekeeping priorities (e.g., suites to clean)  
        - Front desk notes (totals, peak times)  
        - Branding with hotel name and styling  

   k. Connect both Filter Today's Arrivals and Filter Today's Departures → Create Email HTML.

   l. Add **Gmail (Send Daily Staff Report)** node:  
      - Credentials: Gmail OAuth2  
      - To: staff mail (from Add Information)  
      - Subject: Dynamic date string from Add Information node (e.g., "27 July")  
      - Message: HTML from Create Email HTML  

   m. Connect Create Email HTML → Send Daily Staff Report.

6. **Add Sticky Notes** at appropriate positions for block overviews and detailed documentation.

7. **Configure all credentials**:  
   - Google Sheets OAuth2 with access to your reservation spreadsheet  
   - Gmail OAuth2 with permission to send emails on behalf of hotel staff

8. **Test the workflow** with sample reservation data including check-in/out dates, guest emails, and ensure emails are sent only once per guest and daily reports are generated correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 🏨 HOTEL GUEST COMMUNICATION AUTOMATION - Complete guest journey automation from booking to post-stay with email personalization and daily staff reporting. Ideal for hotels, B&Bs, and short-term rentals. | Sticky Note3 content in workflow overview                                                             |
| 🎯 The workflow reduces manual work, front desk inquiries, and missed follow-ups while increasing guest satisfaction and online reviews.                                                                                       | Sticky Note3                                                                                          |
| 📋 Daily staff report includes guest details, room assignments, housekeeping priorities, and statistics, sent every morning at 6:00 AM.                                                                                        | Sticky Note2                                                                                          |
| ⭐ Post-stay review requests include Google and TripAdvisor links, return guest discount codes, and direct contact info to boost reputation and loyalty.                                                                           | Sticky Note1                                                                                          |
| 🏨 Pre-arrival welcome emails sent 1-2 days prior to check-in include reservation details, hotel amenities, and contact info to enhance the guest experience.                                                                    | Sticky Note                                                                                            |
| Gmail nodes require OAuth2 credentials with sending rights. Google Sheets nodes require OAuth2 credentials with read/write access to the reservations spreadsheet.                                                                | Setup instructions                                                                                     |
| Date/time operations use the 'Asia/Kolkata' timezone and 'M/d/yyyy HH:mm:ss' date format consistently throughout the workflow.                                                                                                  | Observed in all date parsing and formatting expressions                                              |
| For modification, update Google Sheets document ID, sheet name, hotel name, staff email, and email templates in Gmail nodes.                                                                                                    | Setup and customization instructions in Sticky Note3                                                 |
| Email HTML templates include inline styling and placeholders for dynamic data using n8n expressions (e.g., `{{ $json['Guest Name'] }}`).                                                                                        | Gmail node message parameter                                                                           |

---

**Disclaimer:** The provided content is derived solely from an n8n automation workflow. All data processed is legal and publicly accessible. The workflow respects all content policies and contains no illegal or offensive elements.