Automatic Event Creation in Google Calendar from Google Sheets Data

https://n8nworkflows.xyz/workflows/automatic-event-creation-in-google-calendar-from-google-sheets-data-3300


# Automatic Event Creation in Google Calendar from Google Sheets Data

### 1. Workflow Overview

This workflow automates the creation of Google Calendar events based on new rows added to a Google Sheet. It is designed to streamline event management by synchronizing event data from a spreadsheet directly into a calendar, eliminating manual entry and reducing errors.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Detects new event entries added to a specified Google Sheet.
- **1.2 Data Formatting:** Processes and formats the raw event data, particularly the event date, to comply with Google Calendar’s requirements.
- **1.3 Event Creation:** Uses the formatted data to create a new event in Google Calendar with customizable attributes such as status and color.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new rows added to a specific Google Sheet, triggering the workflow whenever a new event entry is detected.

- **Nodes Involved:**  
  - New Event Entry Listener

- **Node Details:**

  - **New Event Entry Listener**  
    - **Type:** Google Sheets Trigger  
    - **Technical Role:** Monitors a Google Sheet for new rows added, triggering the workflow on each addition.  
    - **Configuration:**  
      - Trigger Event: `rowAdded` (fires when a new row is appended)  
      - Sheet Name: `Sheet1` (gid=0)  
      - Spreadsheet ID: `1dKjIGmcnQgSEMVuWAAFVDaj_MCBFKBX8hCOk5OH2dK4`  
      - Polling Interval: Every minute  
      - Value Render Option: `FORMULA` (to get the evaluated cell values)  
    - **Expressions/Variables:** None explicitly used here; outputs the entire new row data.  
    - **Input Connections:** None (trigger node)  
    - **Output Connections:** Connects to the "Event Date Formatter" node  
    - **Version Requirements:** Requires Google Sheets API enabled and OAuth2 credentials configured for Google Sheets.  
    - **Potential Failures:**  
      - Authentication errors if OAuth2 token expires or is invalid  
      - API quota limits or connectivity issues  
      - Sheet or document ID changes or access revocation  
    - **Sub-workflow:** None

#### 1.2 Data Formatting

- **Overview:**  
  This block extracts relevant event details from the new row and formats the event start date into ISO format (`YYYY-MM-DD`) compatible with Google Calendar.

- **Nodes Involved:**  
  - Event Date Formatter

- **Node Details:**

  - **Event Date Formatter**  
    - **Type:** Code (JavaScript) Node  
    - **Technical Role:** Processes the raw event data, ensuring the event date includes the current year if missing, and converts it to ISO date format.  
    - **Configuration:**  
      - Custom JavaScript code that:  
        - Retrieves the last item from the input data (the new row)  
        - Extracts fields: `Event Name`, `Event Description`, `Event Start Date`, and `Location`  
        - Checks if the event start date string includes the current year; if not, appends it  
        - Converts the date string to a JavaScript Date object and then to ISO format (YYYY-MM-DD)  
        - Returns a JSON object with formatted fields: `eventName`, `eventDescription`, `startDate`, and `location`  
    - **Key Expressions/Variables:**  
      - `items[items.length - 1].json` to get the latest row  
      - `new Date(startDateString).toISOString().split("T")[0]` for date formatting  
    - **Input Connections:** Receives data from "New Event Entry Listener"  
    - **Output Connections:** Sends formatted data to "Google Calendar Event Creator"  
    - **Version Requirements:** Uses n8n Code node version 2 syntax  
    - **Potential Failures:**  
      - Date parsing errors if the input date format is unexpected or invalid  
      - Missing or malformed fields in the input data  
      - JavaScript runtime errors if input data is empty or undefined  
    - **Sub-workflow:** None

#### 1.3 Event Creation

- **Overview:**  
  This block creates a new event in Google Calendar using the formatted event details, allowing customization of event status and color for better calendar management.

- **Nodes Involved:**  
  - Google Calendar Event Creator

- **Node Details:**

  - **Google Calendar Event Creator**  
    - **Type:** Google Calendar Node  
    - **Technical Role:** Creates a new calendar event with the provided details.  
    - **Configuration:**  
      - Operation: `Create Event`  
      - Calendar: Selected from authenticated Google account (default or user-selected)  
      - Event Fields Mapped:  
        - Summary (Title): `={{ $json.eventName }}`  
        - Description: `={{ $json.eventDescription }}`  
        - Location: `={{ $json.location }}`  
        - Start and End Date: Both set to `={{ $json.startDate }}` (all-day event)  
      - Additional Fields:  
        - `allday`: `yes` (event spans entire day)  
        - `color`: `3` (background color for event)  
        - `showMeAs`: `transparent` (event status set as "Available")  
        - `guestsCanInviteOthers`: `true` (allows guests to invite others)  
    - **Key Expressions/Variables:**  
      - Uses expressions to map JSON fields from the previous node  
    - **Input Connections:** Receives formatted event data from "Event Date Formatter"  
    - **Output Connections:** None (end node)  
    - **Version Requirements:** Requires Google Calendar API enabled and OAuth2 credentials configured for Google Calendar  
    - **Potential Failures:**  
      - Authentication errors or expired tokens  
      - API quota exceeded or connectivity issues  
      - Invalid calendar ID or insufficient permissions  
      - Incorrect date format or missing required fields causing event creation failure  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                  |
|----------------------------|-------------------------|---------------------------------|-----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note                | Sticky Note             | Workflow Title Display           | None                        | None                         | # Automate Event Creation in Google Calendar from Google Sheets                              |
| Sticky Note1               | Sticky Note             | Workflow Description Display     | None                        | None                         | ## Description ... (full detailed description of workflow purpose and benefits)             |
| New Event Entry Listener   | Google Sheets Trigger   | Detects new rows added to Sheet  | None                        | Event Date Formatter          |                                                                                              |
| Event Date Formatter       | Code (JavaScript)       | Formats event data and date      | New Event Entry Listener     | Google Calendar Event Creator |                                                                                              |
| Google Calendar Event Creator | Google Calendar        | Creates event in Google Calendar | Event Date Formatter         | None                         |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it**: "Automate Event Creation in Google Calendar from Google Sheets".

2. **Add the Google Sheets Trigger Node**:  
   - Search and add "Google Sheets Trigger".  
   - Authenticate with your Google account (OAuth2).  
   - Set the Spreadsheet ID to your target Google Sheet (e.g., `1dKjIGmcnQgSEMVuWAAFVDaj_MCBFKBX8hCOk5OH2dK4`).  
   - Set the Sheet Name to monitor (e.g., `Sheet1` or `gid=0`).  
   - Set the Trigger Event to `Row Added`.  
   - Set Polling Interval to every minute.  
   - Set Value Render Option to `FORMULA` to get evaluated cell values.  
   - Save and test the node to ensure it triggers on new rows.

3. **Add a Code Node to Format Event Data**:  
   - Add a "Code" node (JavaScript).  
   - Connect it to the Google Sheets Trigger node.  
   - Paste the following JavaScript code (adapt if needed for your column names and date formats):

```javascript
// Get the last item from the input data
const lastEvent = items[items.length - 1].json;

// Extract relevant fields
const eventName = lastEvent["Event Name"];
const eventDescription = lastEvent["Event Description"];
const currentYear = new Date().getFullYear(); 
const location = lastEvent["Location"];

// Ensure the date includes the year
const formatDateWithYear = (dateStr) => {
    return dateStr.includes(currentYear) ? dateStr : `${dateStr} ${currentYear}`;
};

// Format the start date
const startDateString = formatDateWithYear(lastEvent["Event Start Date"]);

// Convert to JavaScript Date object
const startDate = new Date(startDateString);

// Convert to ISO format (YYYY-MM-DD)
const formattedStartDate = startDate.toISOString().split("T")[0];

// Return the last event's formatted data
return [{
    json: {
        eventName,
        eventDescription,
        startDate: formattedStartDate,
        location: location,
    }
}];
```

   - Save and execute the node to verify output.

4. **Add the Google Calendar Node to Create Event**:  
   - Add a "Google Calendar" node.  
   - Authenticate with your Google Calendar account (OAuth2).  
   - Set Operation to `Create Event`.  
   - Select the target calendar or leave default.  
   - Map the fields using expressions:  
     - Summary: `={{ $json.eventName }}`  
     - Description: `={{ $json.eventDescription }}`  
     - Location: `={{ $json.location }}`  
     - Start: `={{ $json.startDate }}`  
     - End: `={{ $json.startDate }}` (for all-day event)  
   - Under Additional Fields:  
     - Set `allday` to `yes`  
     - Set `color` to `3` (or any preferred color code)  
     - Set `showMeAs` to `transparent` (event status "Available")  
     - Enable `guestsCanInviteOthers` if desired  
   - Connect this node to the Code node.  
   - Save and test by executing the node.

5. **Connect the Nodes in Sequence**:  
   - Google Sheets Trigger → Code Node (Event Date Formatter) → Google Calendar Node (Event Creator).

6. **Test the Workflow**:  
   - Add a new row in your Google Sheet with event details (Event Name, Event Description, Event Start Date, Location).  
   - Wait for the trigger or manually execute the workflow to verify the event is created in Google Calendar with correct details.

7. **Optional Customizations**:  
   - Modify the Code node to handle different date formats or time zones.  
   - Adjust Google Calendar node parameters to set event status as "Busy" (`showMeAs: busy`) or change colors.  
   - Add error handling or notifications for failures.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow was built by the AI development team at WeblineIndia.                                       | https://www.weblineindia.com/ai-development.html                                               |
| Hire AI developers to build custom workflows tailored to your needs.                                     | https://www.weblineindia.com/hire-ai-developers.html                                           |
| Ensure Google Sheets API and Google Calendar API are enabled in Google Cloud Console before use.          | Google Cloud Console API Library                                                               |
| OAuth2 credentials must be configured in n8n for both Google Sheets and Google Calendar nodes.            | n8n Credentials Configuration                                                                  |
| The workflow assumes the Google Sheet columns: Event Name, Event Description, Event Start Date, Location. | Adjust column names in the Code node if your sheet uses different headers.                      |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and AI agents to analyze, reproduce, and customize the automation effectively.