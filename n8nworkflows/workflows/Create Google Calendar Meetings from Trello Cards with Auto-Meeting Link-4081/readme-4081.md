Create Google Calendar Meetings from Trello Cards with Auto-Meeting Link

https://n8nworkflows.xyz/workflows/create-google-calendar-meetings-from-trello-cards-with-auto-meeting-link-4081


# Create Google Calendar Meetings from Trello Cards with Auto-Meeting Link

---
### 1. Workflow Overview

This workflow automates the creation of Google Calendar meetings triggered by Trello card movements. Specifically, when a Trello card is moved to a designated list, the workflow extracts email addresses from the card’s description, creates a Google Calendar event at the card’s due date, invites the extracted attendees, and writes the meeting link back onto the Trello card. This workflow targets free Trello users who want seamless calendar scheduling without leaving Trello.

**Logical Blocks:**

- **1.1 Trigger & Filter:** Detect Trello card movement and filter for the target list.
- **1.2 Trello Card Data Retrieval:** Fetch detailed card info including description and due date.
- **1.3 Email Extraction:** Parse the card description to extract and separate email addresses.
- **1.4 Google Calendar Meeting Creation:** Create a calendar event with attendees and capture the meeting link.
- **1.5 Trello Card Update:** Add the generated meeting link back to the Trello card description or comments.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Filter

- **Overview:** Listens for a Trello card movement event and filters to proceed only if the card is moved to the specific list that should trigger meeting creation.
- **Nodes Involved:**  
  - Trigger Move Card in Trello  
  - Filter Action  

- **Node Details:**

  - **Trigger Move Card in Trello**  
    - *Type:* Trello Trigger  
    - *Role:* Initiates workflow on Trello card movement events via webhook.  
    - *Configuration:* Uses a webhook ID to listen for card move events; no extra parameters configured.  
    - *Inputs:* External Trello webhook event.  
    - *Outputs:* Card movement event data.  
    - *Edge Cases:* Webhook misconfiguration, Trello API downtime, network issues.  
    - *Version:* v1.  
  
  - **Filter Action**  
    - *Type:* Filter  
    - *Role:* Filters node data to continue only if the card is moved to the intended Trello list (exact list ID or name check expected).  
    - *Configuration:* Condition likely checks the destination list ID or name from the trigger's output.  
    - *Inputs:* Output from Trello Trigger node.  
    - *Outputs:* Passes data onward only if condition met, else halts flow.  
    - *Edge Cases:* Incorrect or missing list ID causing false negatives.  
    - *Version:* v2.2.  

#### 2.2 Trello Card Data Retrieval

- **Overview:** Retrieves full card details for the filtered card, including description and due date, necessary for meeting creation.
- **Nodes Involved:**  
  - Trello: Get Card Info  
- **Node Details:**

  - **Trello: Get Card Info**  
    - *Type:* Trello node  
    - *Role:* Fetches detailed card information by card ID provided from previous nodes.  
    - *Configuration:* Uses card ID from the filter output to query Trello API for card data.  
    - *Inputs:* Card ID from filtered event.  
    - *Outputs:* Card detail data including description, due date, and other metadata.  
    - *Edge Cases:* Card deleted or inaccessible, API rate limits, missing due date.  
    - *Version:* v1.

#### 2.3 Email Extraction

- **Overview:** Parses the card description text to extract email addresses, then separates them into a format suitable for Google Calendar attendee invitations.
- **Nodes Involved:**  
  - Get Email (Code)  
  - Separates Emails (Code)  

- **Node Details:**

  - **Get Email**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts raw email address strings from the card description using regex or similar parsing.  
    - *Configuration:* Uses JavaScript code to scan description text for emails.  
    - *Inputs:* Card description from Trello Get Card Info node.  
    - *Outputs:* Raw string or array of matched email addresses.  
    - *Key Expressions:* Regex pattern probably used to match email addresses.  
    - *Edge Cases:* No emails in description, malformed emails, multiple formats.  
    - *Version:* v2.

  - **Separates Emails**  
    - *Type:* Code (JavaScript)  
    - *Role:* Converts raw email string into an array of individual email objects formatted as required by Google Calendar (usually objects with email properties).  
    - *Configuration:* Splits email string, trims whitespace, forms attendee objects.  
    - *Inputs:* Raw email string from Get Email node.  
    - *Outputs:* Array of email attendee objects.  
    - *Edge Cases:* Empty or invalid email entries, duplicates.  
    - *Version:* v2.

#### 2.4 Google Calendar Meeting Creation

- **Overview:** Creates a Google Calendar event at the due date/time of the Trello card and invites the extracted attendees.
- **Nodes Involved:**  
  - Calendar: Create Meeting  

- **Node Details:**

  - **Calendar: Create Meeting**  
    - *Type:* Google Calendar node  
    - *Role:* Creates a calendar event with title, start/end times, and attendees.  
    - *Configuration:* Uses card title as event summary, due date as event start/end, and attendee emails from previous step. Also retrieves the meeting link from the response.  
    - *Inputs:* Attendee list, event timing, and summary from previous nodes.  
    - *Outputs:* Event data including meeting link (e.g., Google Meet link).  
    - *Edge Cases:* Invalid date/time, OAuth token expiration, Google API limits or errors.  
    - *Version:* v1.3.

#### 2.5 Trello Card Update

- **Overview:** Adds the Google Calendar meeting link back to the Trello card, usually appending to the description or adding a comment.
- **Nodes Involved:**  
  - Trello: Add Meeting Link  

- **Node Details:**

  - **Trello: Add Meeting Link**  
    - *Type:* Trello node  
    - *Role:* Updates the Trello card by adding the generated meeting link, ensuring visibility for card collaborators.  
    - *Configuration:* Uses card ID and meeting link URL to update card description or comments.  
    - *Inputs:* Card ID and meeting link from Google Calendar node.  
    - *Outputs:* Confirmation of Trello card update.  
    - *Edge Cases:* Card locked or archived, API failures, invalid meeting link format.  
    - *Version:* v1.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                       | Input Node(s)               | Output Node(s)            | Sticky Note                                    |
|----------------------------|---------------------|------------------------------------|----------------------------|---------------------------|------------------------------------------------|
| Trigger Move Card in Trello | Trello Trigger      | Initiates workflow on card movement | -                          | Filter Action             |                                                |
| Filter Action              | Filter              | Filters for correct Trello list     | Trigger Move Card in Trello | Trello: Get Card Info     |                                                |
| Trello: Get Card Info       | Trello              | Fetches detailed card info          | Filter Action              | Get Email                 |                                                |
| Get Email                  | Code                | Extracts emails from card description| Trello: Get Card Info      | Separates Emails          |                                                |
| Separates Emails           | Code                | Formats emails into attendee objects | Get Email                  | Calendar: Create Meeting  |                                                |
| Calendar: Create Meeting    | Google Calendar     | Creates meeting & retrieves meeting link | Separates Emails          | Trello: Add Meeting Link  |                                                |
| Trello: Add Meeting Link    | Trello              | Adds meeting link back to Trello card | Calendar: Create Meeting   | -                         |                                                |
| Get List ID                | HTTP Request        | (Unused or unconnected in this version) | -                          | -                         |                                                |
| Get Organization ID        | HTTP Request        | (Unused or unconnected in this version) | -                          | -                         |                                                |
| Sticky Note                | Sticky Note         | Various notes (mostly empty)        | -                          | -                         |                                                |
| Sticky Note1               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note2               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note3               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note4               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note5               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note6               | Sticky Note         |                                    | -                          | -                         |                                                |
| Sticky Note7               | Sticky Note         |                                    | -                          | -                         |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Trello Trigger node**  
   - Type: Trello Trigger  
   - Configure webhook to listen for card movement events (e.g., “Card moved to a list”).  
   - Set up Trello credentials with API key and token.  

2. **Add a Filter node**  
   - Connect from Trello Trigger.  
   - Configure to check that the card’s new list ID matches the target list that should trigger meeting creation.  
   - Use an expression to compare the event data’s destination list ID.  

3. **Add a Trello node to get card info**  
   - Connect from Filter node’s “true” output.  
   - Operation: “Get Card”  
   - Use the card ID from the filter output to fetch full card details (especially description and due date).  

4. **Add a Code node to extract emails**  
   - Connect from Trello Get Card Info.  
   - Use JavaScript to parse card description text for emails, e.g.:  
     ```js
     const description = items[0].json.desc || '';
     const regex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-z]{2,}/g;
     const emails = description.match(regex) || [];
     return [{ json: { emails: emails.join(', ') } }];
     ```  
   - Output a comma-separated string or array of emails.  

5. **Add another Code node to format emails for Google Calendar**  
   - Connect from previous Code node.  
   - Split the email string into an array and format each as `{ email: "address" }` for Google Calendar attendees, e.g.:  
     ```js
     const emails = items[0].json.emails.split(',').map(e => e.trim()).filter(e => e);
     const attendees = emails.map(email => ({ email }));
     return [{ json: { attendees } }];
     ```  

6. **Add a Google Calendar node to create the meeting**  
   - Connect from the second Code node.  
   - Operation: Create Event  
   - Use card title for event summary.  
   - Use card due date/time as event start and end (adjust end time as needed, e.g., +30 minutes).  
   - Add attendees from previous node.  
   - Configure Google OAuth2 credentials with calendar scope.  

7. **Add a Trello node to update the card with meeting link**  
   - Connect from Google Calendar node.  
   - Operation: Update Card or Add Comment  
   - Use card ID to update the description or add a comment containing the meeting link (e.g., Google Meet URL from event).  

8. **Test the workflow**  
   - Move a Trello card to the configured list with emails in description and a due date.  
   - Verify calendar event is created and Trello card updated with meeting link.  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Video demonstration of the automation working with free Trello and Google Calendar accounts. | [YouTube Video](https://youtu.be/hzn1dL6HPE8)                                                  |
| Full setup instructions available online.                                                    | [Instructions Page](https://cool-homegrown-029.notion.site/Trello-Free-Google-Calendar-1f2718e8c6ba80af9ed2e96dffcb4c43?pvs=4) |

---

**Disclaimer:**  
The text above is derived exclusively from an n8n workflow automation. It respects all applicable content policies and contains no illegal, offensive, or protected content. All processed data is legal and publicly accessible.