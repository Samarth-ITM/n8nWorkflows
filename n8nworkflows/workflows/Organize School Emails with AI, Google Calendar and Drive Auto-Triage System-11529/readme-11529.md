Organize School Emails with AI, Google Calendar and Drive Auto-Triage System

https://n8nworkflows.xyz/workflows/organize-school-emails-with-ai--google-calendar-and-drive-auto-triage-system-11529


# Organize School Emails with AI, Google Calendar and Drive Auto-Triage System

### 1. Workflow Overview

This workflow, named **"Email Auto-Triage and Organization Hub"**, is designed to automate the processing and organization of school-related emails for parents or caregivers. It leverages AI to classify incoming emails into categories such as schedules, what-to-bring lists, notices, and contact information, then organizes the data across Gmail, Google Calendar, and Google Drive. A daily reminder email consolidates upcoming "what to bring" events to help users prepare in advance.

The workflow is logically divided into three major blocks:

- **1.1 Email Reception and Classification:** Watches for new Gmail messages, fetches full email data, then uses an AI model to extract structured information and classify emails into predefined categories.

- **1.2 Data Storage and Event Creation:** Routes classified emails into branches that save notices and contacts as text files on Google Drive, create calendar events for schedules and what-to-bring notices, and save attached photos to Drive.

- **1.3 Daily Reminder System:** Runs a scheduled trigger each morning to fetch tomorrow’s calendar events, filter those that involve items to bring, and sends a reminder email summarizing these events.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Classification

- **Overview:**  
  This block listens for new incoming school-related emails in Gmail, fetches full message details, and applies an AI model to extract structured data including event info, categories, and contact details.

- **Nodes Involved:**  
  - When Email Arrives  
  - Get a message  
  - Workflow Configuration  
  - Extract Email Info  
  - OpenAI Chat Model  
  - Route by Category  
  - Extract Contacts  
  - Filter  

- **Node Details:**

  - **When Email Arrives**  
    - *Type:* Gmail Trigger  
    - *Role:* Watches Gmail inbox for new emails, triggers workflow every minute.  
    - *Configuration:* No filters specified, polling every minute.  
    - *Inputs:* None (trigger)  
    - *Outputs:* Email metadata (IDs)  
    - *Edge Cases:* Potential Gmail API rate limits; emails without expected metadata.

  - **Get a message**  
    - *Type:* Gmail  
    - *Role:* Fetches full email content by message ID.  
    - *Configuration:* Uses message ID from trigger node, downloads full message including subject and body.  
    - *Inputs:* Email ID from "When Email Arrives"  
    - *Outputs:* Full email JSON including text and snippet.  
    - *Edge Cases:* Missing or deleted emails; Gmail API errors.

  - **Workflow Configuration**  
    - *Type:* Set  
    - *Role:* Stores shared configuration parameters such as Google Drive folder IDs and reminder email address.  
    - *Configuration:* Defines `photosFolderId`, `noticesFolderId`, `contactsFolderId`, and `reminderEmail`.  
    - *Inputs:* From "Get a message"  
    - *Outputs:* Configuration JSON for downstream use.  

  - **Extract Email Info**  
    - *Type:* Langchain Information Extractor (AI)  
    - *Role:* Uses an AI prompt to classify the email into one category (Schedule, What to Bring, Notice, Contacts) and extract structured attributes like event title, date, items to bring, contacts, and attachment presence.  
    - *Configuration:* Custom system prompt instructs detailed extraction rules and output format as raw JSON.  
    - *Inputs:* Email text/plain or snippet from "Get a message"  
    - *Outputs:* Structured JSON with keys: category, hasAttachments, eventTitle, eventDescription, eventDate, itemsToBring, contacts, subject, id.  
    - *Edge Cases:* AI misclassification; missing or ambiguous data; time zone or date parsing errors.

  - **OpenAI Chat Model**  
    - *Type:* Langchain OpenAI Chat  
    - *Role:* Supports the "Extract Email Info" node by providing advanced language model capabilities (GPT-4.1-mini).  
    - *Configuration:* Uses GPT-4.1-mini model preset.  
    - *Inputs:* Text from "Extract Email Info"  
    - *Outputs:* AI-generated classification and extraction JSON.

  - **Route by Category**  
    - *Type:* Switch  
    - *Role:* Routes the processed email data into separate branches based on the extracted category, enabling customized processing for each email type.  
    - *Configuration:* Routes to outputs named Schedule, What to Bring, Notice, Contacts, and fallback Notice for unrecognized cases.  
    - *Inputs:* JSON output from "Extract Email Info"  
    - *Outputs:* Conditional outputs to downstream processing nodes.  
    - *Edge Cases:* Unmatched categories fall to fallback branch; missing category key causes fallback.

  - **Extract Contacts**  
    - *Type:* Set  
    - *Role:* Extracts contact information from the email sender and subject for further processing.  
    - *Configuration:* Assigns `contactInfo`, `emailSubject`, and timestamp `extractedDate`.  
    - *Inputs:* Email JSON from "Extract Email Info"  
    - *Outputs:* JSON with contact data.

  - **Filter**  
    - *Type:* Filter  
    - *Role:* Filters out items without contact info before saving.  
    - *Configuration:* Checks that `contactInfo` is not empty.  
    - *Inputs:* JSON from "Extract Contacts"  
    - *Outputs:* Contacts that have content proceed to saving.

---

#### 2.2 Data Storage and Event Creation

- **Overview:**  
  This block handles storing extracted data in Google Drive and Google Calendar. Notices and contacts are saved as text files, calendar events are created for schedules and what-to-bring categories, and photos attached to emails are saved in Drive.

- **Nodes Involved:**  
  - Save Notice to Drive  
  - Prepare Calendar Event  
  - Add to Calendar  
  - Get Email Attachments  
  - Filter Photos Only  
  - Save Photos to Drive  
  - Save Contacts to Drive  

- **Node Details:**

  - **Save Notice to Drive**  
    - *Type:* Google Drive  
    - *Role:* Saves notice emails as text files in a specified Drive folder.  
    - *Configuration:* Creates text files named `Notice - <eventTitle>.txt` with email details including title, date, category, items to bring, and description. Folder ID is dynamically set from "Workflow Configuration".  
    - *Inputs:* Email JSON from "Route by Category" (Notice output)  
    - *Outputs:* Confirmation of file creation.  
    - *Edge Cases:* Drive API errors, folder permission issues.

  - **Prepare Calendar Event**  
    - *Type:* Set  
    - *Role:* Builds a structured calendar event object with title, description, start and end times, and items to bring.  
    - *Configuration:* Uses extracted data from "Extract Email Info"; event end time defaults to 1 hour after start or 1 day plus 1 hour from now if missing; description includes items to bring in Japanese.  
    - *Inputs:* JSON from "Route by Category" (Schedule or What to Bring outputs)  
    - *Outputs:* Calendar event data for next node.

  - **Add to Calendar**  
    - *Type:* Google Calendar  
    - *Role:* Creates calendar events in Google Calendar with the prepared event data.  
    - *Configuration:* Uses calendar address `fairytale0726@gmail.com`, event start and end ISO strings, and event summary and description from previous node.  
    - *Inputs:* Calendar event data from "Prepare Calendar Event"  
    - *Outputs:* Event creation confirmation.  
    - *Edge Cases:* Calendar API errors, invalid date formats.

  - **Get Email Attachments**  
    - *Type:* Gmail  
    - *Role:* Retrieves attachments from the original email message.  
    - *Configuration:* Downloads all attachments based on email ID; uses message ID from "Get a message".  
    - *Inputs:* JSON from "Route by Category"  
    - *Outputs:* Email attachments as binary data.  
    - *Edge Cases:* Large attachments, unsupported file types, or missing attachments.

  - **Filter Photos Only**  
    - *Type:* Filter  
    - *Role:* Filters to only image attachments based on MIME type containing "image".  
    - *Configuration:* Checks MIME type of first attachment.  
    - *Inputs:* Attachments from "Get Email Attachments"  
    - *Outputs:* Only image attachments proceed.

  - **Save Photos to Drive**  
    - *Type:* Google Drive  
    - *Role:* Saves image attachments to a configured Google Drive folder.  
    - *Configuration:* Saves with original filename to folder ID from "Workflow Configuration".  
    - *Inputs:* Filtered image attachments from "Filter Photos Only"  
    - *Outputs:* Confirmation of saved files.  
    - *Edge Cases:* Drive API quota or permission issues.

  - **Save Contacts to Drive**  
    - *Type:* Google Drive  
    - *Role:* Saves extracted contact information as text files in Drive.  
    - *Configuration:* Creates files named `Contact - <emailSubject>.txt` containing contact info and extraction date; folder ID from "Workflow Configuration".  
    - *Inputs:* Contacts filtered by "Filter" node  
    - *Outputs:* Confirmation of file creation.

---

#### 2.3 Daily Reminder System

- **Overview:**  
  This block runs every morning to fetch all calendar events scheduled for the next day, filters those that include "what to bring" information, and sends a reminder email summarizing these events.

- **Nodes Involved:**  
  - Daily Reminder Check  
  - Get Tomorrow's Events  
  - Filter What to Bring Events  
  - Send Reminder Email  

- **Node Details:**

  - **Daily Reminder Check**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers the workflow at 8:00 AM daily to start the reminder process.  
    - *Configuration:* Fixed time trigger at hour 8 daily.  
    - *Inputs:* None (time trigger)  
    - *Outputs:* Trigger signal.

  - **Get Tomorrow's Events**  
    - *Type:* Google Calendar  
    - *Role:* Retrieves all calendar events scheduled for the entire next day.  
    - *Configuration:* Sets timeMin and timeMax to the start and end of the next day; queries the same calendar address as event creation.  
    - *Inputs:* Trigger from "Daily Reminder Check"  
    - *Outputs:* List of calendar events for tomorrow.

  - **Filter What to Bring Events**  
    - *Type:* Filter  
    - *Role:* Filters events whose descriptions include the Japanese keyword 持ち物 (items to bring) to identify relevant events.  
    - *Configuration:* Checks if event description contains 持ち物.  
    - *Inputs:* Events from "Get Tomorrow's Events"  
    - *Outputs:* Filtered events that require bringing items.

  - **Send Reminder Email**  
    - *Type:* Gmail  
    - *Role:* Sends a reminder email summarizing the next day's events and their items to bring.  
    - *Configuration:* Sends to the email address of the event creator; subject and message body include event summary, start/end times, and description formatted with Japanese reminders.  
    - *Inputs:* Filtered events from "Filter What to Bring Events"  
    - *Outputs:* Email send confirmation.  
    - *Edge Cases:* Email delivery issues; missing creator email.

---

### 3. Summary Table

| Node Name             | Node Type                              | Functional Role                                      | Input Node(s)                  | Output Node(s)                               | Sticky Note                                                                                                          |
|-----------------------|--------------------------------------|-----------------------------------------------------|-------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When Email Arrives     | Gmail Trigger                        | Watches Gmail inbox for new emails                   | -                             | Get a message                                | Step 1 – Capture and classify new school emails                                                                      |
| Get a message          | Gmail                               | Fetches full email details                           | When Email Arrives             | Workflow Configuration                       | Step 1 – Capture and classify new school emails                                                                      |
| Workflow Configuration | Set                                 | Stores shared folder IDs and email address          | Get a message                 | Extract Email Info                           | Step 1 – Capture and classify new school emails                                                                      |
| Extract Email Info     | Langchain Information Extractor     | AI extracts structured info from email text         | Workflow Configuration        | Route by Category, Get Email Attachments, Extract Contacts | Step 1 – Capture and classify new school emails                                                                      |
| OpenAI Chat Model      | Langchain OpenAI Chat               | Provides AI language model for extraction            | Extract Email Info (ai_languageModel) | Extract Email Info                            | Step 1 – Capture and classify new school emails                                                                      |
| Route by Category      | Switch                             | Routes emails by extracted category                  | Extract Email Info            | Save Notice to Drive, Prepare Calendar Event, Save Notice to Drive, Save Notice to Drive | Step 1 – Capture and classify new school emails                                                                      |
| Get Email Attachments  | Gmail                              | Downloads email attachments                           | Route by Category             | Filter Photos Only                           | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Filter Photos Only     | Filter                             | Filters only image attachments                        | Get Email Attachments         | Save Photos to Drive                         | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Save Photos to Drive   | Google Drive                      | Saves image attachments to Drive                      | Filter Photos Only            | -                                            | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Save Notice to Drive   | Google Drive                      | Saves notices as text files                           | Route by Category             | -                                            | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Prepare Calendar Event | Set                               | Builds calendar event data                            | Route by Category             | Add to Calendar                             | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Add to Calendar        | Google Calendar                  | Creates calendar events                               | Prepare Calendar Event        | -                                            | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Extract Contacts       | Set                               | Extracts contact info from emails                     | Route by Category             | Filter                                        | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Filter                 | Filter                            | Filters contacts with non-empty info                  | Extract Contacts              | Save Contacts to Drive                       | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Save Contacts to Drive | Google Drive                      | Saves contact info as text files                      | Filter                       | -                                            | Step 2 – Save notices, events, photos, and contacts                                                                  |
| Daily Reminder Check   | Schedule Trigger                 | Starts daily reminder process                         | -                             | Get Tomorrow's Events                        | Step 3 – Daily reminder for tomorrow’s “What to Bring” events                                                       |
| Get Tomorrow's Events  | Google Calendar                  | Gets all events for the next day                      | Daily Reminder Check          | Filter What to Bring Events                  | Step 3 – Daily reminder for tomorrow’s “What to Bring” events                                                       |
| Filter What to Bring Events | Filter                       | Filters events with “what to bring” info              | Get Tomorrow's Events         | Send Reminder Email                         | Step 3 – Daily reminder for tomorrow’s “What to Bring” events                                                       |
| Send Reminder Email    | Gmail                             | Sends reminder email about tomorrow’s events          | Filter What to Bring Events   | -                                            | Step 3 – Daily reminder for tomorrow’s “What to Bring” events                                                       |
| Sticky Note - Overview | Sticky Note                      | Describes overall workflow purpose                     | -                             | -                                            | See note content in Section 5                                                                                         |
| Sticky Note - Step 1   | Sticky Note                      | Describes Step 1 block                                 | -                             | -                                            | See note content in Section 5                                                                                         |
| Sticky Note - Step 2   | Sticky Note                      | Describes Step 2 block                                 | -                             | -                                            | See note content in Section 5                                                                                         |
| Sticky Note - Step 3   | Sticky Note                      | Describes Step 3 block                                 | -                             | -                                            | See note content in Section 5                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Name: "When Email Arrives"  
   - Type: Gmail Trigger  
   - Configuration: Poll every minute, no filters set.

2. **Create Gmail Get Message Node:**  
   - Name: "Get a message"  
   - Type: Gmail  
   - Configuration: Operation "get", use messageId from "When Email Arrives" node.

3. **Create Set Node for Configuration:**  
   - Name: "Workflow Configuration"  
   - Type: Set  
   - Configuration: Define variables -  
     - photosFolderId = "1mrFvUmNddLow3i1ZIBOSg9r2R3K2L67m"  
     - noticesFolderId = same as photosFolderId  
     - contactsFolderId = same as photosFolderId  
     - reminderEmail = "fairytale0726@gmail.com"

4. **Create Langchain Information Extractor Node:**  
   - Name: "Extract Email Info"  
   - Type: Langchain Information Extractor  
   - Configuration:  
     - Text source: Use plain text, HTML, or snippet from "Get a message"  
     - System prompt: Use detailed prompt to classify email into Schedule, What to Bring, Notice, or Contacts, and extract fields (category, hasAttachments, eventTitle, eventDate, etc.)  
     - Output: Raw JSON only (no explanations).

5. **Create Langchain OpenAI Chat Model Node:**  
   - Name: "OpenAI Chat Model"  
   - Type: Langchain OpenAI Chat  
   - Configuration: Model set to "gpt-4.1-mini"

6. **Connect "OpenAI Chat Model" output to "Extract Email Info" AI input.**

7. **Create Switch Node:**  
   - Name: "Route by Category"  
   - Type: Switch  
   - Configuration:  
     - Rules:  
       - Output "Schedule" if category == "Schedule"  
       - Output "What to Bring" if category == "What to Bring"  
       - Output "extra" if hasAttachments == true  
       - Output "Notice" as fallback  
     - Outputs renamed accordingly.

8. **Create Google Drive Node for Notices:**  
   - Name: "Save Notice to Drive"  
   - Type: Google Drive  
   - Configuration:  
     - Operation: Create from Text  
     - Filename: "Notice - {{eventTitle}}.txt"  
     - Content: Formatted text with eventTitle, eventDate, category, itemsToBring, eventDescription  
     - Folder ID: From Workflow Configuration (noticesFolderId)

9. **Create Set Node to Prepare Calendar Event:**  
   - Name: "Prepare Calendar Event"  
   - Type: Set  
   - Configuration: Assign calendarEventTitle, calendarEventDescription (includes itemsToBring), calendarEventEnd (default +1 hour), calendarEventsStart (eventDate), itemsToBring (from Extract Email Info).

10. **Create Google Calendar Node:**  
    - Name: "Add to Calendar"  
    - Type: Google Calendar  
    - Configuration:  
      - Calendar: "fairytale0726@gmail.com"  
      - Start: calendarEventsStart  
      - End: calendarEventEnd  
      - Summary: calendarEventTitle  
      - Description: calendarEventDescription

11. **Create Gmail Node to Get Attachments:**  
    - Name: "Get Email Attachments"  
    - Type: Gmail  
    - Configuration: Operation "get" with downloadAttachments enabled, use messageId from "Get a message"

12. **Create Filter Node to Filter Photos Only:**  
    - Name: "Filter Photos Only"  
    - Type: Filter  
    - Configuration: Check that first attachment MIME type contains "image"

13. **Create Google Drive Node to Save Photos:**  
    - Name: "Save Photos to Drive"  
    - Type: Google Drive  
    - Configuration:  
      - Operation: Upload  
      - Folder ID: photosFolderId from Workflow Configuration  
      - Filename: Original attachment file name  
      - Input data field: attachment_0

14. **Create Set Node to Extract Contacts:**  
    - Name: "Extract Contacts"  
    - Type: Set  
    - Configuration:  
      - contactInfo: From "When Email Arrives" email From field  
      - emailSubject: From "Get a message" Subject  
      - extractedDate: Current timestamp

15. **Create Filter Node to Validate Contacts:**  
    - Name: "Filter"  
    - Type: Filter  
    - Configuration: Only continue if contactInfo is not empty

16. **Create Google Drive Node to Save Contacts:**  
    - Name: "Save Contacts to Drive"  
    - Type: Google Drive  
    - Configuration:  
      - Operation: Create from Text  
      - Filename: "Contact - {{emailSubject}}.txt"  
      - Content: contactInfo, emailSubject, extractedDate  
      - Folder ID: contactsFolderId from Workflow Configuration

17. **Create Schedule Trigger Node:**  
    - Name: "Daily Reminder Check"  
    - Type: Schedule Trigger  
    - Configuration: Run daily at 8:00 AM

18. **Create Google Calendar Node to Get Tomorrow's Events:**  
    - Name: "Get Tomorrow's Events"  
    - Type: Google Calendar  
    - Configuration:  
      - Calendar: "fairytale0726@gmail.com"  
      - TimeMin: Start of next day  
      - TimeMax: End of next day  
      - Operation: Get all events

19. **Create Filter Node for What to Bring Events:**  
    - Name: "Filter What to Bring Events"  
    - Type: Filter  
    - Configuration: Filter events whose description contains the Japanese word 持ち物 (items to bring)

20. **Create Gmail Node to Send Reminder Email:**  
    - Name: "Send Reminder Email"  
    - Type: Gmail  
    - Configuration:  
      - Send to: Event creator's email  
      - Subject: Event summary or default "明日の予定のお知らせ"  
      - Message: Formatted with event summary, start/end time, description including items to bring

21. **Connect nodes according to logic:**  
    - Trigger → Get message → Workflow Configuration → Extract Email Info → AI model → Route by Category  
    - Route branches: Schedule/What to Bring → Prepare Calendar Event → Add to Calendar  
    - Route → Get Email Attachments → Filter Photos Only → Save Photos to Drive  
    - Route → Extract Contacts → Filter → Save Contacts to Drive  
    - Route → Save Notice to Drive (for Notices and fallback)  
    - Daily Reminder Check → Get Tomorrow's Events → Filter What to Bring Events → Send Reminder Email

22. **Set up credentials:**  
    - Gmail OAuth2 for email nodes  
    - Google Drive OAuth2 for Drive nodes  
    - Google Calendar OAuth2 for calendar nodes  
    - OpenAI API key for Langchain OpenAI Chat model

23. **Test workflow with real school emails to verify correct classification and data saving.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                                                                                                                                                                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Email Auto-Triage and Organization Hub** workflow uses AI to automatically classify school-related emails into categories for easier management and notification. It supports Japanese text and uses specific Japanese keywords (e.g., 持ち物) to detect items to bring. The workflow is ideal for busy parents or caregivers to never miss important school events or items.                                                                                                                                                                   | Sticky Note - Overview                                                                                                                                                                                                                                                                                                                                                                                                                         |
| Users must configure their Gmail, Google Drive, and Google Calendar credentials with OAuth2. Folder IDs for storing photos, notices, and contacts must be set in the "Workflow Configuration" node. The reminder email address is also configured there.                                                                                                                                                                                                                                                    | Sticky Note - Overview / Workflow Configuration node                                                                                                                                                                                                                                                                                                                                                                                          |
| The AI prompt in "Extract Email Info" is carefully designed to output only raw JSON without explanations, facilitating downstream automation. It expects dates in ISO 8601 format; if the year is missing, it infers the next occurrence. Default event start time is 09:00 if only date is given.                                                                                                                                                                                                            | Extract Email Info node detailed prompt                                                                                                                                                                                                                                                                                                                                                                                                        |
| The daily reminder system triggers at 8 AM every day and sends reminders only for events that include "what to bring" in their description, making it practical for preparing school supplies.                                                                                                                                                                                                                                                                                                  | Sticky Note - Step 3                                                                                                                                                                                                                                                                                                                                                                                                                            |
| This workflow can be extended to add further integrations, such as logging to Google Sheets or sending notifications via messaging apps.                                                                                                                                                                                                                                                                                                                                                         | Sticky Note - Overview                                                                                                                                                                                                                                                                                                                                                                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.