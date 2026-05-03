Two-way Property Repair Management System with Google Sheets & Drive

https://n8nworkflows.xyz/workflows/two-way-property-repair-management-system-with-google-sheets---drive-10004


# Two-way Property Repair Management System with Google Sheets & Drive

---

## 1. Workflow Overview

This workflow, titled **"Two-way Property Repair Management System with Google Sheets & Drive"**, automates the management of tenant property repair requests and updates. It is designed to streamline communication and record-keeping between tenants submitting repair requests and building managers overseeing repair progress.

The workflow is divided into two main logical blocks representing two complementary workflows:

- **Workflow 1: New Repair Request Intake**
  - Triggered by new form submissions via Google Sheets.
  - Processes tenant repair requests, generates a unique identifier (UUID) for each unit, formats the data, and sends a notification email to the building manager with a link to update the repair status.

- **Workflow 2: Repair Update Processing**
  - Triggered by repair update submissions (via an n8n form or manual trigger).
  - Retrieves existing repair records by UUID, handles optional photo uploads to Google Drive, merges new update details with existing data, updates the Google Sheet, and optionally sends a confirmation email.

Each workflow is composed of multiple nodes grouped into functional blocks such as input reception, data formatting, conditional branching, file handling, data merging, spreadsheet updates, and email notifications.

---

## 2. Block-by-Block Analysis

### 2.1 Workflow 1: New Repair Request Intake

#### Overview
This block handles new tenant repair requests received via a Google Sheets trigger. It extracts and formats the incoming data, generates a UUID, updates the spreadsheet row with this UUID, and sends an alert email to the building manager containing repair details and a link for updates.

#### Nodes Involved
- New repair form submitted (Google Sheets Trigger)
- Isolate new entry (Code)
- Format Data (Set)
- Add UUID # (Google Sheets update)
- Report Alert + Info (Gmail)
- Sticky Notes (various explanatory notes)

#### Node Details

- **New repair form submitted**
  - Type: Google Sheets Trigger
  - Role: Triggers workflow on new row added to Google Sheet (repair request log)
  - Configuration: Polls every minute on specified spreadsheet and sheet.
  - Input: New row data from repair request form.
  - Outputs: Full row item.
  - Edge cases: Delay in polling; data format mismatch.

- **Isolate new entry**
  - Type: Code
  - Role: Extracts the last (most recent) item from trigger data.
  - Configuration: JavaScript returns only the last item.
  - Inputs: Multiple rows (if any)
  - Outputs: Single latest row item.
  - Edge cases: Empty input array (should handle gracefully).

- **Format Data**
  - Type: Set
  - Role: Maps and formats raw form data into standardized fields, builds compound fields such as UUID.
  - Key expressions:
    - UUID: Combines building address (uppercase, no spaces) and unit number (trimmed).
    - Copies various form fields into standardized JSON keys.
  - Inputs: Single row JSON
  - Outputs: Formatted JSON with consistent field names.
  - Edge cases: Missing form fields, inconsistent data formats.

- **Add UUID #**
  - Type: Google Sheets (Update)
  - Role: Updates the same spreadsheet row by adding the generated UUID using the Timestamp as the matching column.
  - Configuration: Uses Timestamp column to locate row; writes Unique Unit ID.
  - Inputs: Formatted data with UUID.
  - Outputs: Updated spreadsheet row data.
  - Edge cases: Matching row not found, permissions error.

- **Report Alert + Info**
  - Type: Gmail (Send Email)
  - Role: Sends a formatted notification email to the building manager with repair request details and a link to update repair status.
  - Configuration: Uses Gmail credential; email content is HTML formatted with dynamic expressions for request fields.
  - Inputs: Formatted data including UUID and contact info.
  - Outputs: Email sent confirmation.
  - Edge cases: Auth failure, sending limits, invalid email addresses.

- **Sticky Notes**
  - Provide instructions and explanations related to node setup, credentials, and workflow logic.
  - Not executable.

---

### 2.2 Workflow 2: Repair Update Processing

#### Overview
This block processes repair update submissions, including optional photo uploads. It retrieves existing repair data by UUID, conditionally handles photos by uploading them to Google Drive, merges new update details with existing data, updates the spreadsheet row, and optionally sends a confirmation email.

#### Nodes Involved
- REPLACE w/ N8N FORM trigger (Manual Trigger placeholder)
- If (Conditional)
- Get row(s) in sheet (Google Sheets)
- Existing Details (Set)
- Separate Photo Files (Code)
- Photo Uploading (SplitInBatches)
- Combine Photo URLs (Code)
- New details (Set)
- Merge Data (Merge)
- Update w/ Repair (Google Sheets Update)
- Send a message (Gmail)
- Upload Repair Photo (Google Drive)
- Sticky Notes (various)

#### Node Details

- **REPLACE w/ N8N FORM trigger**
  - Type: Manual Trigger (placeholder)
  - Role: Trigger for repair update submissions, intended to be replaced by an n8n Form Trigger.
  - Inputs: External form submission.
  - Outputs: Initiates workflow on new update.
  - Edge cases: Manual trigger requires replacement for automation.

- **If**
  - Type: If
  - Role: Branch logic to check if repairPhotos field exists in incoming data.
  - Condition: Checks existence of 'repairPhotos' property.
  - Outputs:
    - True: No photos submitted, proceed with new details.
    - False: Photos submitted, proceed to photo processing.
  - Edge cases: Missing or malformed 'repairPhotos' field.

- **Get row(s) in sheet**
  - Type: Google Sheets (Read)
  - Role: Fetches existing repair data row using UUID as lookup key.
  - Configuration: Returns first match by Unique Unit ID column.
  - Inputs: UUID from update form.
  - Outputs: Existing repair record as JSON.
  - Edge cases: No matching row found, permissions error.

- **Existing Details**
  - Type: Set
  - Role: Extracts key existing data fields from fetched sheet row for merging.
  - Inputs: Row JSON from Get row(s) in sheet.
  - Outputs: JSON with UUID, Action, nameDate, repairPhotos for merging.
  - Edge cases: Missing expected fields.

- **Separate Photo Files**
  - Type: Code
  - Role: Splits binary photo data into individual file items for uploading.
  - Inputs: Binary photo data from form submission.
  - Outputs: Array of single-photo items with original file names and mime types.
  - Edge cases: No photos or unexpected binary data structure.

- **Photo Uploading**
  - Type: SplitInBatches
  - Role: Processes photo uploads one by one to Google Drive folder.
  - Inputs: Individual photo files from Separate Photo Files.
  - Outputs: URL info of uploaded files.
  - Edge cases: Upload failures, API rate limits.

- **Combine Photo URLs**
  - Type: Code
  - Role: Aggregates all uploaded photo URLs into a single string separated by commas.
  - Inputs: List of uploaded photo webViewLinks.
  - Outputs: JSON with combined string and array of URLs.
  - Edge cases: Duplicate URLs, empty uploads.

- **New details**
  - Type: Set
  - Role: Creates JSON with new update fields including UUID, Action, nameDate, and combined photo URLs.
  - Inputs: Output from Combine Photo URLs and form submission.
  - Outputs: Prepared new update JSON.
  - Edge cases: Missing photo URLs or other fields.

- **Merge Data**
  - Type: Merge
  - Role: Combines existing details and new update details keyed by UUID, preserving all fields.
  - Configuration: Combine mode, keep everything, match on UUID.
  - Inputs: Existing Details and New details.
  - Outputs: Combined JSON ready for spreadsheet update.
  - Edge cases: Mismatched UUIDs, missing fields.

- **Update w/ Repair**
  - Type: Google Sheets (Update)
  - Role: Updates the repair request row with merged data including appended actions, names, and photo URLs.
  - Configuration: Matches on Unique Unit ID; appends new values to existing spreadsheet columns.
  - Inputs: Merged JSON.
  - Outputs: Updated spreadsheet row.
  - Edge cases: Concurrent edits, missing match row.

- **Send a message**
  - Type: Gmail (Send Email)
  - Role: Optional email notification about repair update, including a link to the repair log.
  - Configuration: Gmail credential, HTML content with dynamic fields.
  - Inputs: Updated spreadsheet data.
  - Outputs: Email sent confirmation.
  - Edge cases: Email send failures.

- **Upload Repair Photo**
  - Type: Google Drive (Upload)
  - Role: Uploads individual repair photos to a specified Google Drive folder with a naming convention based on UUID and date.
  - Configuration: Uses folder ID for storage, input data from photo files.
  - Inputs: Individual photo binary data.
  - Outputs: Upload response with webViewLink.
  - Edge cases: Upload failures, permissions.

- **Sticky Notes**
  - Provide setup instructions, usage notes, and reminders about replacing manual triggers and matching field names.

---

## 3. Summary Table

| Node Name                     | Node Type              | Functional Role                              | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                                                     |
|-------------------------------|------------------------|----------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| New repair form submitted      | Google Sheets Trigger  | Trigger on new repair request row             |                              | Isolate new entry             | # Workflow 1: Behind the Scenes: Tenant fills form; triggers processing.                                                                                        |
| Isolate new entry              | Code                   | Extract last new entry for processing          | New repair form submitted     | Format Data                  |                                                                                                                                                                |
| Format Data                   | Set                    | Format and standardize form data; create UUID | Isolate new entry             | Report Alert + Info, Add UUID# | ## Sheets: Google Sheets Credential                                                                                                                            |
| Add UUID #                   | Google Sheets (Update)  | Add unique UUID to the sheet row               | Format Data                  |                              | ## Sheets: Add UUID to row; use TIMESTAMP as match column                                                                                                      |
| Report Alert + Info           | Gmail                  | Send email alert with repair request details  | Format Data                  |                              | ## Alert Email: Email Credential; includes link to update repair                                                                                               |
| REPLACE w/ N8N FORM trigger.  | Manual Trigger          | Placeholder for repair update form submission  |                              | If, Get row(s) in sheet       | ## REPLACE THIS TRIGGER WITH N8N FORM = Repair Update Form                                                                                                     |
| If                           | If                     | Check if repairPhotos field exists             | REPLACE w/ N8N FORM trigger  | Separate Photo Files, New details |                                                                                                                                                                |
| Get row(s) in sheet           | Google Sheets (Read)    | Retrieve existing repair record by UUID        | If (True branch)             | Existing Details             | ## Sheets: Use UUID to collect row info                                                                                                                        |
| Existing Details              | Set                    | Extract existing repair data fields            | Get row(s) in sheet          | Merge Data                   |                                                                                                                                                                |
| Separate Photo Files          | Code                   | Split binary photo data into individual files  | If (False branch)            | Photo Uploading              | ## Google Drive: Add Drive Credential; select folder for photo storage                                                                                         |
| Photo Uploading               | SplitInBatches          | Upload photos one by one to Google Drive       | Separate Photo Files          | Combine Photo URLs, Upload Repair Photo |                                                                                                                                                                |
| Upload Repair Photo           | Google Drive (Upload)   | Upload individual repair photo to Drive folder | Photo Uploading              | Photo Uploading              |                                                                                                                                                                |
| Combine Photo URLs            | Code                   | Combine all photo URLs into one string          | Photo Uploading              | New details                  | ## Combine Photo URL(s): combine URLs into single key for spreadsheet                                                                                          |
| New details                  | Set                    | Prepare new repair update data including photos | Combine Photo URLs           | Merge Data                   |                                                                                                                                                                |
| Merge Data                   | Merge                  | Merge new update data with existing repair data | Existing Details, New details | Update w/ Repair            |                                                                                                                                                                |
| Update w/ Repair             | Google Sheets (Update)  | Update spreadsheet row with merged repair info | Merge Data                  | Send a message               | ## Sheets: Use *TIMESTAMP* as column to match on; field names must align with form fields                                                                       |
| Send a message               | Gmail                  | Send optional email notification about update | Update w/ Repair            |                              | ## Optional Email: Email Credential; for developer or manager notifications                                                                                     |
| Sticky Notes (multiple)       | Sticky Note             | Provide instructions, explanations, setup info | Various                      |                              | Includes detailed workflow explanations, setup instructions, and important reminders such as replacing manual triggers and matching field names in sheet setup |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps in n8n to build the entire workflow:

### Workflow 1: New Repair Request Intake

1. **Create Google Sheets Trigger**
   - Node: Google Sheets Trigger
   - Credential: Add Google Sheets OAuth2 credential.
   - Document: Select your "Repair Request Log" spreadsheet.
   - Sheet: Select the sheet where form responses are stored (e.g., "Form Responses 7").
   - Trigger event: "Row Added"
   - Poll interval: Every minute.

2. **Add Code Node "Isolate new entry"**
   - Purpose: Extract the last item from incoming data.
   - JavaScript:
     ```js
     const last = items[items.length - 1];
     return [last];
     ```

3. **Add Set Node "Format Data"**
   - Map form fields to standardized names.
   - Create UUID by combining building address and unit number:
     - `uniqueUnitId = BuildingAddress.replace(/\s+/g, '').toUpperCase() + '-' + UnitNumber.trim()`
   - Map other form fields like Timestamp, Name, Urgency, Contact Info, Category, Explanation, Photos/Videos, Feedback/Suggestions.

4. **Add Google Sheets Update Node "Add UUID #"**
   - Credential: Use same Google Sheets OAuth2 credential.
   - Document & Sheet: Same as trigger.
   - Operation: Update row.
   - Match Column: Timestamp (from formatted data).
   - Columns to update: Unique Unit ID with the generated UUID.

5. **Add Gmail Node "Report Alert + Info"**
   - Credential: Setup Gmail OAuth2 credential.
   - Subject: e.g., "n8nTESt - Urgency: {{urgency}}. Service Request @ {{building}}, Unit {{unit}}"
   - Message: HTML formatted with all relevant fields from formatted data.
   - Include a link to an n8n form for repair updates (replace placeholder with actual form URL).
   - Connect input from Format Data node.

6. **Connect nodes sequentially** as:
   - Google Sheets Trigger → Isolate new entry → Format Data → [parallel] Report Alert + Info and Add UUID #

7. **Add Sticky Notes** for setup documentation and credential reminders.

---

### Workflow 2: Repair Update Processing

1. **Add Trigger Node**
   - Replace the placeholder "Manual Trigger" with an n8n Form Trigger node.
   - Configure form fields to match the repair update form.
   - This node triggers on new repair update form submissions.

2. **Add If Node "If"**
   - Condition: Check if the incoming JSON contains the field `repairPhotos`.
   - If TRUE (no photos), proceed with new details.
   - If FALSE (photos present), proceed to process photos.

3. **Add Google Sheets Read Node "Get row(s) in sheet"**
   - Credential: Google Sheets OAuth2.
   - Document & Sheet: Repair Request Log.
   - Operation: Lookup row matching the UUID from the form submission.
   - Return first match.

4. **Add Set Node "Existing Details"**
   - Extract key fields from the fetched row: UUID, Action, Name/Date, Repair Photos.

5. **Add Code Node "Separate Photo Files"**
   - JavaScript to split binary photo data into individual file items.
   - Handles file names and mime types properly.

6. **Add SplitInBatches Node "Photo Uploading"**
   - Processes photos one by one for upload.

7. **Add Google Drive Upload Node "Upload Repair Photo"**
   - Credential: Google Drive OAuth2.
   - Folder: Select folder for storing repair photos.
   - File name: Use expression combining UUID and nameDate for naming convention.
   - Input: Binary photo files from previous node.

8. **Add Code Node "Combine Photo URLs"**
   - Combine uploaded file URLs into a comma-separated string.
   - Outputs combined URLs and an array.

9. **Add Set Node "New details"**
   - Assemble new update fields, including UUID, Action, nameDate, and combined photo URLs.

10. **Add Merge Node "Merge Data"**
    - Mode: Combine.
    - Join Mode: keep everything.
    - Match on UUID.
    - Inputs: Existing Details and New details.

11. **Add Google Sheets Update Node "Update w/ Repair"**
    - Credential: Google Sheets OAuth2.
    - Document & Sheet: Repair Request Log.
    - Operation: Update row matching Unique Unit ID.
    - Map columns to append new data to existing cells (e.g., concatenate Action, Name/Date, Repair Photos).

12. **Add Gmail Node "Send a message" (Optional)**
    - Credential: Gmail OAuth2.
    - Email content: Notify about repair update with relevant details.
    - Connect from Update w/ Repair node.

13. **Connect nodes as follows:**
    - Trigger → If
    - If (True) → Get row(s) in sheet → Existing Details → Merge Data
    - If (False) → Separate Photo Files → Photo Uploading → Upload Repair Photo → Photo Uploading → Combine Photo URLs → New details → Merge Data
    - Merge Data → Update w/ Repair → Send a message (optional)

14. **Add Sticky Notes** throughout for setup, reminders on replacing triggers, folder selections, and field name matching.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This template automates tenant repair requests and updates via Google Forms and Google Sheets. Two workflows handle new submissions and manager updates, ensuring all repair data stays in a single, organized row. | Workflow Introduction Sticky Note                                                                                     |
| PRODUCTION URL must be included in the "Repair Request" email at the end of Workflow 1 for update links to function correctly.                                         | Sticky Note on repair update submission email                                                                        |
| Google Drive setup requires adding Drive credentials and selecting the folder for photo storage.                                                                       | Sticky Note near photo upload nodes                                                                                   |
| Google Sheets credential required with correct permissions; UUID field must be added to the spreadsheet and used as key for lookups and updates.                      | Sticky Notes near Google Sheets nodes                                                                                 |
| Field names in Google Sheets must exactly match form field names for proper data mapping and updates.                                                                 | Sticky Notes explaining field name importance                                                                         |
| The "REPLACE w/ N8N FORM trigger" node must be replaced with an actual n8n form trigger node to automate repair update submissions.                                   | Sticky Note warning before the manual trigger                                                                         |
| Repair photos are uploaded individually to Google Drive, and their URLs are combined into a single string to be stored in one spreadsheet cell for easy reference.     | Sticky Note explaining photo URL combination                                                                          |
| Email nodes use Gmail OAuth2 credentials and are optional for notification to building managers or tenants.                                                           | Sticky Notes describing purpose and setup of email nodes                                                             |
| For best reliability, set polling intervals and API limits according to expected load and Google API quotas.                                                         | General best practice note                                                                                             |
| This workflow requires Google OAuth2 credentials for Sheets and Drive and Gmail OAuth2 credentials for email notifications.                                            | Credential setup reminders                                                                                            |
| For detailed step-by-step instructions, review the sticky notes embedded throughout the workflow.                                                                      | Sticky notes with setup instructions                                                                                   |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation platform. This processing strictly respects content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---