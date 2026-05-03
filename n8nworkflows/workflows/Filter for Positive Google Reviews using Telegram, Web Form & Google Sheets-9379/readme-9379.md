Filter for Positive Google Reviews using Telegram, Web Form & Google Sheets

https://n8nworkflows.xyz/workflows/filter-for-positive-google-reviews-using-telegram--web-form---google-sheets-9379


# Filter for Positive Google Reviews using Telegram, Web Form & Google Sheets

### 1. Workflow Overview

This workflow is designed to manage and filter customer feedback collected via Telegram, a web form, and Google Sheets. Its main purpose is to encourage customers to leave positive Google reviews by sending them incentive offers and follow-up reminders while capturing private, potentially negative feedback for internal handling.

The workflow is logically divided into these blocks:

- **1.1 Initial Interaction and Customer Data Capture:** Receive user messages from Telegram, check if the user is a returning customer, and update their details in Google Sheets.
- **1.2 Incentive Offer Delivery and Status Update:** Send a discount offer to new or eligible customers and update their status in the sheet accordingly.
- **1.3 Feedback Collection via Webhook:** Receive feedback submissions from a web form webhook and save private feedback messages to Google Sheets.
- **1.4 Link Click Tracking and Status Update:** Track when a user clicks the review link and update their status in Google Sheets.
- **1.5 Scheduled Follow-up Reminders:** Periodically check customers who have been contacted but not completed the review process, and send follow-up reminder messages.
- **1.6 Delay and Review Link Delivery:** Introduce a delay after the initial offer before sending the actual review page link to ensure a better user experience.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Interaction and Customer Data Capture

- **Overview:**  
  This block handles incoming Telegram messages from users, checks if they already exist in the Google Sheet database, and updates or adds their details as needed.

- **Nodes Involved:**  
  - Initial User Message (Telegram Trigger)  
  - Fetch Rows from sheet (Google Sheets)  
  - checks if customer exists in sheets (IF)  
  - Updates customer details (Google Sheets)  
  - checks if they still didn't claim the offer (IF)  
  - Already claimed offer msg (Telegram)  

- **Node Details:**  

  - **Initial User Message**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point for user messages received on Telegram.  
    - *Config:* Listens for message updates only. Uses Telegram credentials.  
    - *Inputs:* Telegram messages.  
    - *Outputs:* Passes Telegram message data to next nodes.  
    - *Failures:* Telegram API issues, webhook misconfiguration.  

  - **Fetch Rows from sheet**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Checks if the Telegram user ID exists in the Google Sheet (Sheet1) by filtering on the "ID" column.  
    - *Config:* Filters rows where "ID" equals the Telegram user ID (from message.from.id).  
    - *Inputs:* Telegram message data from previous node.  
    - *Outputs:* User data from Google Sheets if found.  
    - *Failures:* Google Sheets API errors, credential issues, empty results if new user.  

  - **checks if customer exists in sheets**  
    - *Type:* IF node  
    - *Role:* Determines if the user is new or returning based on whether an ID was found in the sheet.  
    - *Config:* Checks if the "ID" field is empty.  
    - *Inputs:* Output from Fetch Rows from sheet.  
    - *Outputs:* Branches to update details if new, or check offer claim status if existing.  
    - *Failures:* Expression evaluation errors.  

  - **Updates customer details**  
    - *Type:* Google Sheets (Append)  
    - *Role:* Adds new user details to the sheet with initial status "pending."  
    - *Config:* Appends a new row with ID, first name, last name, bot flag, status, and timestamp.  
    - *Inputs:* User details from Telegram message.  
    - *Outputs:* Triggers sending incentive offer.  
    - *Failures:* Google Sheets API errors, rate limits.  

  - **checks if they still didn't claim the offer**  
    - *Type:* IF node  
    - *Role:* For returning users, checks if their status is "contacted," "clicked," or "Follow-up Sent" to determine if they already claimed the offer or not.  
    - *Config:* OR condition on the "Status" field.  
    - *Inputs:* Existing user data from sheet.  
    - *Outputs:* Sends message if offer already claimed or resends incentive offer.  
    - *Failures:* Expression errors, missing or unexpected status values.  

  - **Already claimed offer msg**  
    - *Type:* Telegram node (Send message)  
    - *Role:* Sends a message to users who have already claimed the offer, preventing duplicate incentives.  
    - *Config:* Personalized message using user's first name.  
    - *Inputs:* Telegram chat ID.  
    - *Outputs:* None (end of flow for this branch).  
    - *Failures:* Telegram API issues.  

---

#### 2.2 Incentive Offer Delivery and Status Update

- **Overview:**  
  This block sends an initial discount offer message to the user and updates their status in Google Sheets to "contacted".

- **Nodes Involved:**  
  - Send Incentive Offer (Telegram)  
  - Updates Status (Google Sheets)  
  - 2 hours delay for feedback (Wait)  

- **Node Details:**  

  - **Send Incentive Offer**  
    - *Type:* Telegram Send Message  
    - *Role:* Sends a personalized discount offer message to the user via Telegram.  
    - *Config:* Text includes user's first name and a 5% discount offer. Uses Telegram chat ID.  
    - *Inputs:* Telegram user data.  
    - *Outputs:* Triggers status update node.  
    - *Failures:* Telegram API errors, invalid chat ID.  

  - **Updates Status**  
    - *Type:* Google Sheets (Append or Update)  
    - *Role:* Updates the user's status to "contacted" in the sheet, matching by ID.  
    - *Config:* Matches on "ID" column, updates "Status" field.  
    - *Inputs:* Telegram user ID from previous node.  
    - *Outputs:* Triggers 2-hour wait node.  
    - *Failures:* Google Sheets API errors, update conflicts.  

  - **2 hours delay for feedback**  
    - *Type:* Wait node  
    - *Role:* Waits 2 hours before sending the review page link to allow time for the user to experience the service.  
    - *Config:* Wait of 2 hours.  
    - *Inputs:* Triggered from status update completion.  
    - *Outputs:* Triggers sending the review page link.  
    - *Failures:* Workflow interruptions can cause delay failure.  

---

#### 2.3 Feedback Collection via Webhook

- **Overview:**  
  This block handles incoming POST requests from a feedback web form, extracts user feedback, and updates the Google Sheet with private feedback messages.

- **Nodes Involved:**  
  - Feedback message (Webhook)  
  - Save Private Feedback to Sheet (Google Sheets)  

- **Node Details:**  

  - **Feedback message**  
    - *Type:* Webhook (HTTP POST)  
    - *Role:* Entry point to receive feedback data from the web form.  
    - *Config:* POST endpoint with unique path, expects JSON body containing rating, userId, and feedback text.  
    - *Inputs:* HTTP requests from the review webpage.  
    - *Outputs:* Passes feedback data to Google Sheets update node.  
    - *Failures:* HTTP errors, data format issues, unauthorized requests.  

  - **Save Private Feedback to Sheet**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the corresponding user's row with the feedback message text, matched on userId.  
    - *Config:* Matches on "ID" column and updates "Feedback Message".  
    - *Inputs:* Feedback data from webhook node.  
    - *Outputs:* None (end of flow for feedback saving).  
    - *Failures:* Google Sheets API errors, missing ID match.  

---

#### 2.4 Link Click Tracking and Status Update

- **Overview:**  
  This block tracks when a user clicks the link to the review page and updates their status to "clicked" in Google Sheets.

- **Nodes Involved:**  
  - Link Click Track (Webhook)  
  - Update Status to 'Clicked' (Google Sheets)  

- **Node Details:**  

  - **Link Click Track**  
    - *Type:* Webhook (HTTP POST)  
    - *Role:* Receives notifications when a user clicks the review link, passing userId.  
    - *Config:* POST endpoint with unique path, expects JSON body with userId.  
    - *Inputs:* Webhook calls triggered by link clicks on the review webpage.  
    - *Outputs:* Triggers status update node.  
    - *Failures:* HTTP errors, malformed requests.  

  - **Update Status to 'Clicked'**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the user status to "clicked" in the sheet matched on userId.  
    - *Config:* Matches on "ID" column, updates "Status" to "clicked".  
    - *Inputs:* UserId from webhook.  
    - *Outputs:* None (end of flow for link click tracking).  
    - *Failures:* Google Sheets API errors, ID mismatch.  

---

#### 2.5 Scheduled Follow-up Reminders

- **Overview:**  
  Runs every 5 minutes to check if any customers with "contacted" status haven't clicked the link within 23 hours, then sends a reminder message and updates their status.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet1 (Google Sheets)  
  - Loop Over Items (Split in Batches)  
  - Timestamp to Boolean (Set)  
  - if 23 hours passed (IF)  
  - Send Review Link Reminder (Telegram)  
  - update status to follow up sent (Google Sheets)  

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger (every 5 minutes)  
    - *Role:* Periodically initiates the follow-up reminder process.  
    - *Config:* Interval trigger set for every few minutes.  
    - *Inputs:* None.  
    - *Outputs:* Triggers retrieval of contacted users.  
    - *Failures:* Scheduler misconfiguration.  

  - **Get row(s) in sheet1**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Retrieves all rows with status "contacted" for follow-up checks.  
    - *Config:* Filter on "Status" = "contacted".  
    - *Inputs:* Trigger from Schedule node.  
    - *Outputs:* List of contacted users.  
    - *Failures:* API errors.  

  - **Loop Over Items**  
    - *Type:* Split in Batches  
    - *Role:* Processes each contacted user individually for status checks.  
    - *Config:* Default batch options.  
    - *Inputs:* Array of contacted users.  
    - *Outputs:* Single item per batch for condition evaluation.  
    - *Failures:* Batch processing issues.  

  - **Timestamp to Boolean**  
    - *Type:* Set node  
    - *Role:* Calculates if 23 hours have passed since the user's timestamp to determine follow-up eligibility.  
    - *Config:* Sets boolean "is_deadline_in_past" by comparing timestamp + 23 hours to current time.  
    - *Inputs:* User data from batch.  
    - *Outputs:* Boolean for IF condition.  
    - *Failures:* Date parsing errors.  

  - **if 23 hours passed**  
    - *Type:* IF node  
    - *Role:* Checks if boolean "is_deadline_in_past" is true to decide if follow-up should happen.  
    - *Config:* Condition on boolean field.  
    - *Inputs:* Output from Set node.  
    - *Outputs:* Triggers reminder message if true.  
    - *Failures:* Expression evaluation errors.  

  - **Send Review Link Reminder**  
    - *Type:* Telegram Send Message  
    - *Role:* Sends a personalized reminder message with review link to users who haven't clicked the link yet.  
    - *Config:* Inline keyboard button with dynamic URL containing user ID.  
    - *Inputs:* User data with chat ID.  
    - *Outputs:* Triggers status update.  
    - *Failures:* Telegram API errors.  

  - **update status to follow up sent**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates user status to "Follow-up Sent" after reminder is sent.  
    - *Config:* Updates by matching phone number or ID.  
    - *Inputs:* User data from previous node.  
    - *Outputs:* Loops back to batch processing for next user.  
    - *Failures:* API errors, incorrect matching columns.  

---

#### 2.6 Delay and Review Link Delivery

- **Overview:**  
  After the 2-hour wait, sends the unique review page link to the user to collect their star rating.

- **Nodes Involved:**  
  - Send Review Page Link (Telegram)  

- **Node Details:**  

  - **Send Review Page Link**  
    - *Type:* Telegram Send Message  
    - *Role:* Sends a personalized message with a clickable button linking to the review webpage containing a unique user ID parameter.  
    - *Config:* Inline keyboard with "Rate Your Visit" button linking to "Your webpage URL/?userId=TelegramChatId".  
    - *Inputs:* Telegram chat ID from initial message.  
    - *Outputs:* None.  
    - *Failures:* Telegram API errors, invalid URLs.  

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                                      | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------------|-------------------------|-----------------------------------------------------|-----------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| Initial User Message           | Telegram Trigger        | Receives Telegram user message                       | -                           | Fetch Rows from sheet           |                                                                                                        |
| Fetch Rows from sheet          | Google Sheets           | Checks if user exists in sheet                       | Initial User Message         | checks if customer exists in sheets |                                                                                                        |
| checks if customer exists in sheets | IF                    | Determines if user is new or returning               | Fetch Rows from sheet        | Updates customer details, checks if they still didn't claim the offer |                                                                                                        |
| Updates customer details       | Google Sheets           | Adds new user details                                | checks if customer exists    | Send Incentive Offer            |                                                                                                        |
| checks if they still didn't claim the offer | IF                    | Checks if user already claimed offer                 | checks if customer exists    | Already claimed offer msg, Send Incentive Offer |                                                                                                        |
| Already claimed offer msg      | Telegram                | Sends message if offer already claimed               | checks if they still didn't claim the offer | -                              |                                                                                                        |
| Send Incentive Offer           | Telegram                | Sends discount offer message                         | Updates customer details     | Updates Status                 |                                                                                                        |
| Updates Status                | Google Sheets           | Updates user status to "contacted"                   | Send Incentive Offer         | 2 hours delay for feedback      |                                                                                                        |
| 2 hours delay for feedback     | Wait                    | Waits 2 hours before sending review link             | Updates Status              | Send Review Page Link          |                                                                                                        |
| Send Review Page Link          | Telegram                | Sends review page link with unique user parameter    | 2 hours delay for feedback   | -                              | Sticky Note9: Dynamic Link with userId parameter                                                      |
| Feedback message              | Webhook                 | Receives feedback from web form                       | -                           | Save Private Feedback to Sheet | Sticky Note7: Flow sends feedback from webpage to sheets                                              |
| Save Private Feedback to Sheet | Google Sheets           | Saves private feedback message                        | Feedback message             | -                              |                                                                                                        |
| Link Click Track              | Webhook                 | Receives link click notifications                     | -                           | Update Status to 'Clicked'      | Sticky Note8: Updates status to "clicked" if link opened                                              |
| Update Status to 'Clicked'     | Google Sheets           | Updates user status to "clicked"                      | Link Click Track             | -                              |                                                                                                        |
| Schedule Trigger             | Schedule Trigger        | Triggers periodic follow-up check                    | -                           | Get row(s) in sheet1           | Sticky Note10: Checks every 5 minutes if 23 hours passed to send follow-up                            |
| Get row(s) in sheet1          | Google Sheets           | Gets all users with status "contacted"               | Schedule Trigger             | Loop Over Items                |                                                                                                        |
| Loop Over Items              | Split In Batches        | Processes contacted users one by one                 | Get row(s) in sheet1         | Timestamp to Boolean           |                                                                                                        |
| Timestamp to Boolean          | Set                     | Checks if 23 hours passed since timestamp             | Loop Over Items              | if 23 hours passed             |                                                                                                        |
| if 23 hours passed           | IF                      | Decides if follow-up message should be sent          | Timestamp to Boolean         | Send Review Link Reminder      |                                                                                                        |
| Send Review Link Reminder     | Telegram                | Sends reminder message with review link              | if 23 hours passed           | update status to follow up sent |                                                                                                        |
| update status to follow up sent | Google Sheets           | Updates status to "Follow-up Sent"                    | Send Review Link Reminder    | Loop Over Items                |                                                                                                        |
| Sticky Note                   | Sticky Note             | Title and section labels                              | -                           | -                              | Various explanatory sticky notes throughout the workflow                                              |
| Sticky Note1                  | Sticky Note             | Title for link click tracking                         | -                           | -                              |                                                                                                        |
| Sticky Note2                  | Sticky Note             | Title for feedback to Google Sheets                   | -                           | -                              |                                                                                                        |
| Sticky Note3                  | Sticky Note             | Blank (placeholder or spacing)                        | -                           | -                              |                                                                                                        |
| Sticky Note4                  | Sticky Note             | Workflow credits and contact info                     | -                           | -                              |                                                                                                        |
| Sticky Note5                  | Sticky Note             | Detailed instructions and overview                     | -                           | -                              |                                                                                                        |
| Sticky Note6                  | Sticky Note             | Note on incentive strategy                             | -                           | -                              |                                                                                                        |
| Sticky Note7                  | Sticky Note             | Notes that feedback flows into Sheets                  | -                           | -                              |                                                                                                        |
| Sticky Note8                  | Sticky Note             | Notes on link click status update                      | -                           | -                              |                                                                                                        |
| Sticky Note9                  | Sticky Note             | Explanation about dynamic review link with userId     | -                           | -                              |                                                                                                        |
| Sticky Note10                 | Sticky Note             | Explanation about 23 hours follow-up logic             | -                           | -                              |                                                                                                        |
| Sticky Note11                 | Sticky Note             | Telegram Bot creation instructions                      | -                           | -                              |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Credentials:**  
   - Set up a Telegram bot via BotFather, get API token, and add credentials in n8n.

2. **Create Google Sheets Credentials:**  
   - Ensure Google Sheets API access and configure credentials in n8n.

3. **Prepare Google Sheet:**  
   - Create a sheet with columns: ID, First Name, Last Name, Status, Feedback Message, Timestamp.  
   - Copy the Sheet ID from the URL.

4. **Create Node: Initial User Message**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates only.  
   - Credential: Use Telegram credentials.  

5. **Create Node: Fetch Rows from sheet**  
   - Type: Google Sheets (Read)  
   - Parameters: Filter rows where "ID" equals `{{ $json.message.from.id.toString() }}`.  
   - Sheet Name: gid=0 (Sheet1).  
   - Document ID: Paste your Google Sheet ID.  
   - Credential: Google Sheets credentials.  
   - Connect from: Initial User Message.

6. **Create Node: checks if customer exists in sheets**  
   - Type: IF node  
   - Condition: Check if `{{ $json.ID.toString() }}` is empty.  
   - Connect from: Fetch Rows from sheet.

7. **Create Node: Updates customer details**  
   - Type: Google Sheets (Append)  
   - Parameters: Append row with fields:  
     - ID: `{{ $('Initial User Message').item.json.message.from.id }}`  
     - Is Bot: `{{ $('Initial User Message').item.json.message.from.is_bot }}`  
     - Status: "pending"  
     - Last Name: `{{ $('Initial User Message').item.json.message.from.last_name }}`  
     - Timestamp: `{{ $now }}`  
     - First Name : `{{ $('Initial User Message').item.json.message.from.first_name }}`  
   - Sheet Name and Document ID as above.  
   - Connect from: IF node "true" branch (new user).

8. **Create Node: checks if they still didn't claim the offer**  
   - Type: IF node  
   - Condition: OR check if Status equals "contacted", "clicked", or "Follow-up Sent".  
   - Connect from: IF node "false" branch (existing user).

9. **Create Node: Already claimed offer msg**  
   - Type: Telegram Send Message  
   - Parameters:  
     - Text: `"Sorry {{ $('Initial User Message').item.json.message.chat.first_name }}, You have already Claimed this Offer."`  
     - Chat ID: `{{ $('Initial User Message').item.json.message.chat.id }}`  
   - Connect from: IF node "true" branch (already claimed).

10. **Create Node: Send Incentive Offer**  
    - Type: Telegram Send Message  
    - Parameters:  
      - Text: `"Congratulations {{ $('Initial User Message').item.json.message.chat.first_name }}, Here's Your discount of 5%. To claim this show this message to the receptionist."`  
      - Chat ID: `{{ $('Initial User Message').item.json.message.chat.id }}`  
    - Connect from: Updates customer details and IF "false" branch from claim check.

11. **Create Node: Updates Status**  
    - Type: Google Sheets (Append or Update)  
    - Parameters:  
      - Match column: "ID"  
      - Update "Status" to "contacted" for the user ID.  
    - Connect from: Send Incentive Offer.

12. **Create Node: 2 hours delay for feedback**  
    - Type: Wait node  
    - Parameters: Wait for 2 hours.  
    - Connect from: Updates Status.

13. **Create Node: Send Review Page Link**  
    - Type: Telegram Send Message  
    - Parameters:  
      - Text: `"Hi {{ $('Send Incentive Offer').item.json.result.chat.first_name }}, Thank you again for your visit today. Could you please take a moment to rate your experience out of 5 stars? It's just one tap and helps us a lot."`  
      - Chat ID: `{{ $('Initial User Message').item.json.message.chat.id }}`  
      - Reply Markup: Inline keyboard with button text "Rate Your Visit" linking to `https://YourWebpageURL/?userId={{ $('Initial User Message').item.json.message.chat.id }}`  
    - Connect from: 2 hours delay.

14. **Create Node: Feedback message**  
    - Type: Webhook (HTTP POST)  
    - Parameters: Set unique path (e.g., "b3ac4af4-a8fc-43ba-a2ef-be1de79fb2f1").  
    - Connect no inputs (entry node).

15. **Create Node: Save Private Feedback to Sheet**  
    - Type: Google Sheets (Update)  
    - Parameters:  
      - Match on "ID" with `{{ $json.body.userId }}`  
      - Update "Feedback Message" with `{{ $('Feedback message').item.json.body.feedback }}`  
    - Connect from: Feedback message.

16. **Create Node: Link Click Track**  
    - Type: Webhook (HTTP POST)  
    - Parameters: Set unique path (e.g., "366b2145-1d50-45b1-84c0-25bf8765c953").  
    - Connect no inputs (entry node).

17. **Create Node: Update Status to 'Clicked'**  
    - Type: Google Sheets (Update)  
    - Parameters:  
      - Match on "ID" with `{{ $json.body.userId }}`  
      - Update "Status" to "clicked"  
    - Connect from: Link Click Track.

18. **Create Node: Schedule Trigger**  
    - Type: Schedule Trigger  
    - Parameters: Trigger every 5 minutes or desired interval.

19. **Create Node: Get row(s) in sheet1**  
    - Type: Google Sheets (Read)  
    - Parameters: Filter where "Status" = "contacted".  
    - Connect from: Schedule Trigger.

20. **Create Node: Loop Over Items**  
    - Type: Split In Batches  
    - Connect from: Get row(s) in sheet1.

21. **Create Node: Timestamp to Boolean**  
    - Type: Set node  
    - Parameters:  
      - Set boolean "is_deadline_in_past" as `new Date(new Date($json.Timestamp).getTime() + (23 * 60 * 60 * 1000)) < new Date()`  
    - Connect from: Loop Over Items.

22. **Create Node: if 23 hours passed**  
    - Type: IF node  
    - Condition: Check if "is_deadline_in_past" is true.  
    - Connect from: Timestamp to Boolean.

23. **Create Node: Send Review Link Reminder**  
    - Type: Telegram Send Message  
    - Parameters:  
      - Text: `"Hi {{ $('Loop Over Items').item.json['First Name '] }}, This is just a friendly reminder that we'd love to get your star rating on your recent visit. It only takes a second and helps us a lot!"`  
      - Chat ID: `{{ $('Loop Over Items').item.json.ID }}`  
      - Inline keyboard button linking to `https://YourWebpageURL/?userId={{ $('Loop Over Items').item.json.ID }}`  
    - Connect from: IF "true" branch.

24. **Create Node: update status to follow up sent**  
    - Type: Google Sheets (Update)  
    - Parameters:  
      - Match on "Phone Number" (or relevant unique column)  
      - Update "Status" to "Follow-up Sent"  
    - Connect from: Send Review Link Reminder.

25. **Connect update status to follow up sent back to Loop Over Items**  
    - This creates a loop processing remaining batch items.

26. **Add Sticky Notes for documentation and explanation** as per workflow (optional but recommended).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow acts as a personal reputation manager, filtering customer feedback and encouraging positive Google reviews.                                                                                                          | Sticky Note5 content describing overall workflow purpose.                                             |
| Ensure your Google Sheet has exact columns: ID, First Name, Last Name, Status, Feedback Message, Timestamp.                                                                                                                        | Sticky Note5 instructions.                                                                            |
| The review link contains a dynamic `userId` parameter to track individual users and link clicks.                                                                                                                                  | Sticky Note9 explanation.                                                                              |
| Telegram bot creation steps: Use BotFather, create bot, copy API token, add to n8n credentials.                                                                                                                                     | Sticky Note11 instructions.                                                                            |
| Follow-up messages are sent after 23 hours if users haven't clicked the review link, optimizing free message windows in platforms like WhatsApp.                                                                                   | Sticky Note10 explanation.                                                                             |
| The workflow uses a free 5% discount offer to incentivize customers to leave reviews; this can be replaced with any other strategy.                                                                                                 | Sticky Note6.                                                                                          |
| For any questions or support, contact: anirudh.n.aeran@gmail.com; explore more tips on LinkedIn: https://www.linkedin.com/in/anirudh-narayan-a-/                                                                                   | Sticky Note4 credits and contacts.                                                                    |
| The web form feedback submission endpoint must be configured to send JSON with `userId`, `rating`, and `feedback` fields to the webhook node URL as POST requests.                                                                 | Webhook node expected input format.                                                                   |
| Review webpage hosting recommendation: Use the provided GitHub repo [Code](https://github.com/anirudhaeran/Google-Review-Feedback-Form) and deploy on Netlify (free).                                                                | Sticky Note5 and linked tutorial video.                                                               |

---

This documentation provides a detailed understanding, node-by-node analysis, and instructions to rebuild the entire workflow in n8n, supporting both advanced users and AI agents in maintenance, troubleshooting, and enhancements.