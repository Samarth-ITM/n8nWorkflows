Create an event file and send it as an email attachment

https://n8nworkflows.xyz/workflows/create-an-event-file-and-send-it-as-an-email-attachment-1083


# Create an event file and send it as an email attachment

### 1. Workflow Overview

This workflow enables the creation of a calendar event file (.ics) and sends it as an email attachment. It is designed for scenarios where users want to automate sending event invitations via email with an attached calendar invite. The workflow consists of three logical blocks:

- **1.1 Manual Trigger**: Initiation of the workflow manually.
- **1.2 Event File Creation**: Generation of a calendar event file (.ics) using the iCalendar node.
- **1.3 Email Sending**: Dispatching the created event file as an email attachment to a recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

**Overview:**  
This block starts the workflow execution manually, allowing users to run the workflow on demand.

**Nodes Involved:**  
- On clicking 'execute'

**Node Details:**  

- **Node Name:** On clicking 'execute'  
- **Type and Technical Role:** Manual Trigger node; triggers the workflow manually.  
- **Configuration Choices:** No parameters configured; default manual trigger settings.  
- **Key Expressions or Variables:** None.  
- **Input and Output Connections:** No input connections; output connected to the iCalendar node.  
- **Version-specific Requirements:** Compatible with n8n version 0.100.0 and later.  
- **Edge Cases/Potential Failures:** None inherent; manual execution depends on user interaction.  
- **Sub-workflow Reference:** None.

---

#### 1.2 Event File Creation

**Overview:**  
This block creates an iCalendar event file (.ics) representing a specific event with defined start and end times, title, and optional additional fields.

**Nodes Involved:**  
- iCalendar

**Node Details:**  

- **Node Name:** iCalendar  
- **Type and Technical Role:** iCal node; generates a calendar event file in .ics format.  
- **Configuration Choices:**  
  - Start: June 11, 2021, 15:30 UTC  
  - End: June 11, 2021, 16:15 UTC  
  - Title: "n8n Community Meetup"  
  - Additional Fields: None configured (empty).  
- **Key Expressions or Variables:** Static datetime strings and event title; no dynamic expressions used.  
- **Input and Output Connections:** Input from Manual Trigger node; output connected to Send Email node.  
- **Version-specific Requirements:** Requires n8n version supporting the iCal node (introduced around v0.120.0).  
- **Edge Cases/Potential Failures:**  
  - Incorrect date/time formats could cause errors.  
  - Missing required fields might lead to invalid .ics files.  
  - Timezone considerations are not configured; all times are in UTC by default.  
- **Sub-workflow Reference:** None.

---

#### 1.3 Email Sending

**Overview:**  
This block sends an email with the generated calendar event file attached, allowing recipients to receive the event invite directly in their inbox.

**Nodes Involved:**  
- Send Email

**Node Details:**  

- **Node Name:** Send Email  
- **Type and Technical Role:** Email Send node; sends an email with optional attachments.  
- **Configuration Choices:**  
  - Subject: "n8n Community Meetup 🚀"  
  - Body Text: Personalized invitation message to "Harshil" including event details and attachment notice.  
  - Attachments: Set to pass the incoming data (the .ics file) as an attachment.  
  - SMTP Credential: Uses "Outlook Burner Credentials" for authentication.  
- **Key Expressions or Variables:** Static text in email body; attachment passed dynamically from previous node output.  
- **Input and Output Connections:** Input from iCalendar node; no output connections (end of workflow).  
- **Version-specific Requirements:** Compatible with n8n versions supporting the Email Send node and SMTP credential management.  
- **Edge Cases/Potential Failures:**  
  - SMTP authentication failure if credentials are invalid or expired.  
  - Attachment not properly formatted if the iCalendar node fails or outputs incorrectly.  
  - Email delivery issues due to network or server problems.  
- **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role           | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                 |
|-----------------------|-----------------------|--------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger        | Start workflow manually  | —                      | iCalendar              |                                                                                             |
| iCalendar             | iCal                  | Create event file (.ics) | On clicking 'execute'   | Send Email             | **iCalendar node:** This node will create an event file.                                   |
| Send Email            | Email Send            | Send email with attachment| iCalendar              | —                      | **Send Email:** This node will send the event file as an attachment.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "On clicking 'execute'".  
   - No configuration needed. This node will trigger the workflow manually.

2. **Create iCalendar Node**  
   - Add an **iCal** node named "iCalendar".  
   - Set the **Start** date/time to `"2021-06-11T15:30:00.000Z"` (UTC).  
   - Set the **End** date/time to `"2021-06-11T16:15:00.000Z"` (UTC).  
   - Set the **Title** to `"n8n Community Meetup"`.  
   - Leave **Additional Fields** empty.  
   - Connect the output of "On clicking 'execute'" to the input of this "iCalendar" node.

3. **Create Send Email Node**  
   - Add an **Email Send** node named "Send Email".  
   - Set the **Subject** to `"n8n Community Meetup 🚀"`.  
   - Set the **Text** body to:  
     ```
     Hey Harshil,

     We are excited to invite you to the n8n community meetup!

     With this email you will find the invite attached.

     Looking forward to seeing you at the meetup!

     Cheers,
     Harshil
     ```  
   - In **Attachments**, set the field to `"data"`, which passes the incoming iCalendar file as attachment.  
   - Configure **SMTP credentials**:  
     - Use an SMTP credential named `"Outlook Burner Credentials"` (set up separately with valid SMTP server details, username, and password).  
   - Connect the output of "iCalendar" node to the input of "Send Email" node.

4. **Finalize Connections**  
   - Verify the connections:  
     - Manual Trigger → iCalendar → Send Email.

5. **Test the Workflow**  
   - Execute the "On clicking 'execute'" node manually.  
   - Confirm that the email is sent with the .ics file attached.  
   - Check the recipient inbox and verify the calendar event file is intact and opens correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                         |
|----------------------------------------------------------------------------------------------------|---------------------------------------|
| The iCalendar node generates standard .ics files compatible with most calendar applications.       | Workflow functionality                 |
| Email sending requires valid SMTP credentials; ensure credentials are tested to avoid delivery failure. | SMTP configuration best practices     |
| For timezone-sensitive events, consider enhancing the iCalendar node configuration to specify timezones explicitly. | Timezone importance                    |
| The workflow can be extended to dynamically generate event details based on input data or parameters. | Workflow extensibility                  |
| Screenshot of the workflow is available for visual reference (fileId:498).                          | Workflow visual documentation          |

---

This detailed documentation enables a clear understanding of the workflow structure, logic, and configuration, facilitating easy reproduction, modification, and troubleshooting.