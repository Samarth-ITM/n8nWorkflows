Route Tally affiliate leads to Google Sheets and notify teams in Slack

https://n8nworkflows.xyz/workflows/route-tally-affiliate-leads-to-google-sheets-and-notify-teams-in-slack-13369


# Route Tally affiliate leads to Google Sheets and notify teams in Slack

# 1. Workflow Overview

This workflow receives affiliate lead submissions from a Tally form, routes each lead to an affiliate-specific Google Sheet, creates that sheet if it does not already exist, and then notifies a team in Slack.

Its primary use case is affiliate lead intake where each affiliate has its own spreadsheet named from an affiliation code. The workflow ensures submissions are normalized, stored consistently, and immediately surfaced to internal teams.

## 1.1 Input Reception and Field Normalization
The workflow starts from a Tally trigger and extracts/cleans the relevant form fields. It standardizes name, email, affiliation code, and submission timestamp values before any storage logic occurs.

## 1.2 Affiliate Sheet Discovery and Branching
Using the affiliation code, the workflow searches a specific Google Drive folder for a spreadsheet named `<affiliation_code>_Submissions`. It then branches depending on whether a matching file exists.

## 1.3 Existing Sheet Path
If the spreadsheet already exists, the lead is appended directly to that Google Sheet.

## 1.4 New Sheet Creation Path
If no spreadsheet exists, the workflow creates a new Google Sheets file, moves it into the target Google Drive folder, formats a row payload, and appends the lead as the first data row.

## 1.5 Slack Notification
After either append path succeeds, the workflow builds a Slack Block Kit message and posts it to a configured Slack channel.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Field Normalization

### Overview
This block listens for new Tally form submissions and converts raw webhook-style answers into a cleaner internal structure. It also normalizes formatting so downstream sheet naming and notifications are consistent.

### Nodes Involved
- Submission Trigger
- Extract Form Fields

### Node Details

#### Submission Trigger
- **Type and technical role:** `n8n-nodes-tallyforms.tallyTrigger`  
  Event trigger node that starts the workflow when a Tally form receives a new submission.
- **Configuration choices:**
  - Configured for a specific Tally form ID: `ODajpY`
  - Uses Tally API credentials
- **Key expressions or variables used:**
  - No dynamic expressions in configuration
- **Input and output connections:**
  - Entry point node
  - Outputs to **Extract Form Fields**
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires a functioning Tally integration and webhook registration through n8n
- **Edge cases or potential failure types:**
  - Invalid or expired Tally credentials
  - Form ID changed or deleted in Tally
  - Webhook registration issues if n8n base URL is misconfigured
  - Missing expected answers if the Tally form structure changes
- **Sub-workflow reference:**
  - None

#### Extract Form Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Maps and transforms raw Tally submission fields into a structured JSON payload used everywhere else.
- **Configuration choices:**
  - Assigns:
    - `Name`
    - `Email Address`
    - `Phone Number`
    - `Affiliation Code`
    - `Submission Date`
    - `Submission Time`
  - Keeps transformation logic inside expressions rather than in a code node
- **Key expressions or variables used:**
  - Name normalization:  
    `{{ $json.question_W54VPJ.trim().replace(/\s+/g, ' ').replace(/\b\w/g, c => c.toUpperCase()) }}`
  - Email lowercasing:  
    `{{ $json.question_aYWq09.toLowerCase() }}`
  - Phone passthrough:  
    `{{ $json.question_62qERe }}`
  - Affiliation code normalization:  
    `{{ $json.question_72QroL.replace(/\s+/g, '').toLowerCase() }}`
  - Submission date formatting:  
    `{{ $json.createdAt.toDateTime().format('MM/dd/yyyy') }}`
  - Submission time formatting:  
    `{{ $json.createdAt.toDateTime().format('HH:mm') }}`
- **Input and output connections:**
  - Input from **Submission Trigger**
  - Output to **Find Affiliate Sheet**
- **Version-specific requirements:**
  - Uses Set node version `3.4`
  - Relies on n8n expression support for `toDateTime()` and `format()`
- **Edge cases or potential failure types:**
  - If Tally question IDs change, expressions break silently or return undefined
  - `trim()` / `replace()` may fail if a field is null rather than a string
  - `createdAt` missing or malformed would break date formatting
  - Email lowercasing assumes the email field exists
- **Sub-workflow reference:**
  - None

---

## 2.2 Affiliate Sheet Discovery and Branching

### Overview
This block searches Google Drive for an affiliate-specific spreadsheet inside a fixed folder and decides whether to use the existing file or create a new one.

### Nodes Involved
- Find Affiliate Sheet
- Sheet Exists?

### Node Details

#### Find Affiliate Sheet
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Searches for a file in Google Drive that matches the expected affiliate spreadsheet name.
- **Configuration choices:**
  - Resource: file/folder
  - Search scope:
    - Drive: `My Drive`
    - Folder: `Affiliate_Submissions` via folder ID `1H5x2r0sV0xJGbaZXW-lDjZTUmiZtefv0`
    - Search target: files
  - Returns all matches
  - Requests only fields `name` and `id`
  - Query string is dynamically built from affiliation code plus `_Submissions`
  - `alwaysOutputData: true` ensures downstream branching still receives an item even if no file is found
- **Key expressions or variables used:**
  - Search query:  
    `{{ $json["Affiliation Code"] + "_Submissions" }}`
- **Input and output connections:**
  - Input from **Extract Form Fields**
  - Output to **Sheet Exists?**
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
  - Requires Google Drive OAuth2 credentials with permission to list/search files in the target folder
- **Edge cases or potential failure types:**
  - Invalid Drive credentials
  - Folder ID incorrect or inaccessible
  - Search may return multiple files with the same name; node returns all matches
  - Downstream logic assumes a usable file item structure
  - If no results are returned, `alwaysOutputData` helps preserve flow but item content may be mostly empty
- **Sub-workflow reference:**
  - None

#### Sheet Exists?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the Google Drive search output includes a `name` key, treating that as evidence a file exists.
- **Configuration choices:**
  - Condition checks whether the array of JSON keys contains `"name"`
  - True branch = existing sheet found
  - False branch = create new sheet
- **Key expressions or variables used:**
  - Left value:  
    `{{ $json.keys() }}`
  - Condition: array contains `"name"`
- **Input and output connections:**
  - Input from **Find Affiliate Sheet**
  - True output to **Append Data (Existing Sheet)**
  - False output to **Create Affiliate Sheet**
- **Version-specific requirements:**
  - Uses If node version `2.2`
- **Edge cases or potential failure types:**
  - If Google Drive returns partial/empty data in an unexpected shape, existence detection may be inaccurate
  - If multiple files are returned, this logic does not disambiguate which one should be used
  - A file with `name` but not actually being a Google Sheet would still pass
- **Sub-workflow reference:**
  - None

---

## 2.3 Existing Sheet Path

### Overview
When an affiliate-specific spreadsheet already exists, this block appends the normalized lead directly to that file and named sheet tab.

### Nodes Involved
- Append Data (Existing Sheet)

### Node Details

#### Append Data (Existing Sheet)
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a new lead row to an existing spreadsheet discovered via Google Drive.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document ID taken from Drive search result `id`
  - Sheet name taken from Drive search result `name`
  - Fields mapped explicitly:
    - `Name`
    - `Phone Number`
    - `Email Address`
    - `Submission Date`
    - `Submission Time`
  - Mapping mode: define below
- **Key expressions or variables used:**
  - `Name`: `{{ $('Extract Form Fields').item.json.Name }}`
  - `Phone Number`: `{{ $('Extract Form Fields').item.json["Phone Number"] }}`
  - `Email Address`: `{{ $('Extract Form Fields').item.json["Email Address"] }}`
  - `Submission Date`: `{{ $('Extract Form Fields').item.json["Submission Date"] }}`
  - `Submission Time`: `{{ $('Extract Form Fields').item.json["Submission Time"] }}`
  - Sheet name: `{{ $json.name }}`
  - Document ID: `{{ $json.id }}`
- **Input and output connections:**
  - Input from **Sheet Exists?** true branch
  - Output to **Build Slack Message**
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.7`
  - Requires spreadsheet access via Google Sheets OAuth2
- **Edge cases or potential failure types:**
  - Spreadsheet exists but sheet tab name does not exactly match file name
  - Header row in sheet does not match configured columns
  - File found in Drive is not a valid Google Sheets spreadsheet
  - Google API permission errors or append quota issues
  - If multiple Drive results exist, appending may happen to more than one item unless search output is constrained externally
- **Sub-workflow reference:**
  - None

---

## 2.4 New Sheet Creation Path

### Overview
If no affiliate spreadsheet exists, this block creates one, moves it into the target Drive folder, prepares the row payload, and appends the lead.

### Nodes Involved
- Create Affiliate Sheet
- Move to Target Folder
- Format Row Data
- Append Data (New Sheet)

### Node Details

#### Create Affiliate Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Creates a new Google spreadsheet with an initial sheet tab named after the affiliation code.
- **Configuration choices:**
  - Resource: spreadsheet
  - Spreadsheet title: `<affiliation_code>_Submissions`
  - Creates one sheet tab with the same title
- **Key expressions or variables used:**
  - Title:  
    `{{ $('Extract Form Fields').item.json["Affiliation Code"] + "_Submissions" }}`
  - Initial sheet title:  
    `{{ $('Extract Form Fields').item.json["Affiliation Code"] + "_Submissions" }}`
- **Input and output connections:**
  - Input from **Sheet Exists?** false branch
  - Output to **Move to Target Folder**
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.7`
- **Edge cases or potential failure types:**
  - Duplicate title may still be allowed by Google, creating ambiguity
  - Permission failures for spreadsheet creation
  - Affiliation code containing unsupported characters for sheet names
  - If normalized code is empty, spreadsheet title becomes invalid or unhelpful
- **Sub-workflow reference:**
  - None

#### Move to Target Folder
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Moves the newly created spreadsheet into the designated Google Drive folder used for affiliate submissions.
- **Configuration choices:**
  - Operation: `move`
  - File ID from newly created spreadsheet’s `spreadsheetId`
  - Destination folder fixed to `Affiliate_Submissions`
  - Drive set to `My Drive`
- **Key expressions or variables used:**
  - File ID:  
    `{{ $json.spreadsheetId }}`
- **Input and output connections:**
  - Input from **Create Affiliate Sheet**
  - Output to **Format Row Data**
- **Version-specific requirements:**
  - Uses Google Drive node version `3`
- **Edge cases or potential failure types:**
  - Spreadsheet created successfully but move fails due to Drive permissions
  - Invalid folder ID
  - Cross-drive restrictions if file and folder are in different drive scopes
- **Sub-workflow reference:**
  - None

#### Format Row Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds a clean single-item JSON object whose keys exactly match the target sheet columns.
- **Configuration choices:**
  - JavaScript returns one item with:
    - `Name`
    - `Email Address`
    - `Phone Number`
    - `Submission Date`
    - `Submission Time`
  - Pulls values from the first item of **Extract Form Fields**
- **Key expressions or variables used:**
  - Uses:
    - `$('Extract Form Fields').first().json.Name`
    - `$('Extract Form Fields').first().json["Email Address"]`
    - `$('Extract Form Fields').first().json["Phone Number"]`
    - `$('Extract Form Fields').first().json["Submission Date"]`
    - `$('Extract Form Fields').first().json["Submission Time"]`
- **Input and output connections:**
  - Input from **Move to Target Folder**
  - Output to **Append Data (New Sheet)**
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If upstream data is missing, code returns undefined values
  - If the workflow processes multiple items unexpectedly, `.first()` may ignore later items
- **Sub-workflow reference:**
  - None

#### Append Data (New Sheet)
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the formatted lead row to the newly created spreadsheet and its first sheet tab.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet ID from **Create Affiliate Sheet**
  - Sheet identified by sheet ID from the first tab in `sheets[0].properties.sheetId`
  - Mapping mode: auto-map input data
  - Expected columns:
    - Name
    - Email Address
    - Phone Number
    - Submission Date
    - Submission Time
  - Cell format set to `RAW`
- **Key expressions or variables used:**
  - Sheet ID:  
    `{{ $('Create Affiliate Sheet').item.json.sheets[0].properties.sheetId }}`
  - Spreadsheet ID:  
    `{{ $('Create Affiliate Sheet').item.json.spreadsheetId }}`
- **Input and output connections:**
  - Input from **Format Row Data**
  - Output to **Build Slack Message**
- **Version-specific requirements:**
  - Uses Google Sheets node version `4.7`
- **Edge cases or potential failure types:**
  - Brand-new sheet may not yet contain matching headers, depending on n8n behavior and API expectations
  - If auto-mapping cannot find the expected columns, row insertion may fail or produce empty cells
  - Referencing `sheets[0]` assumes the created spreadsheet has at least one sheet
- **Sub-workflow reference:**
  - None

---

## 2.5 Slack Notification

### Overview
This block generates a Slack Block Kit payload based on the lead data and posts it to a configured Slack channel. It runs after either the existing-sheet or new-sheet append path completes.

### Nodes Involved
- Build Slack Message
- Send Lead Notification

### Node Details

#### Build Slack Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Constructs a Slack `blocks` array for a visually formatted message.
- **Configuration choices:**
  - Builds:
    - Header block
    - Name section
    - Email section
    - Optional phone section
    - Affiliate attribution section
  - Omits the phone block when the phone number is an empty string
- **Key expressions or variables used:**
  - Name: `$input.first().json.Name`
  - Email: `$input.first().json["Email Address"]`
  - Phone: `$input.first().json["Phone Number"]`
  - Affiliate code: `$('Extract Form Fields').first().json["Affiliation Code"]`
- **Input and output connections:**
  - Input from:
    - **Append Data (Existing Sheet)**
    - **Append Data (New Sheet)**
  - Output to **Send Lead Notification**
- **Version-specific requirements:**
  - Uses Code node version `2`
- **Edge cases or potential failure types:**
  - If append nodes return items without the expected lead fields, message content may be blank
  - Optional phone logic only checks for empty string, not null/undefined
  - Slack block formatting errors can occur if payload structure is altered incorrectly
- **Sub-workflow reference:**
  - None

#### Send Lead Notification
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends the constructed Slack Block Kit message to a specific channel.
- **Configuration choices:**
  - Message type: `block`
  - Channel selected by ID: `C0A2JFJ3L3Z`
  - Blocks payload bound directly from incoming JSON
  - Slack credential uses a bot/app connection
- **Key expressions or variables used:**
  - `blocksUi`: `{{ $json }}`
  - `text` is effectively empty (`=`)
- **Input and output connections:**
  - Input from **Build Slack Message**
  - No downstream nodes
- **Version-specific requirements:**
  - Uses Slack node version `2.3`
  - Requires bot token with appropriate posting permissions such as `chat:write`
- **Edge cases or potential failure types:**
  - Slack app not invited to the target channel
  - Invalid channel ID
  - Expired/revoked Slack credentials
  - Block payload malformed
  - Some Slack workspaces require fallback text even for block messages
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Submission Trigger | Tally Trigger | Receives new Tally form submissions |  | Extract Form Fields | ## Capture Form Submissions<br>Triggers the workflow instantly when a new lead submits the Tally form and extracts their data. |
| Extract Form Fields | Set | Normalizes and maps raw Tally fields into internal lead fields | Submission Trigger | Find Affiliate Sheet | ## Capture Form Submissions<br>Triggers the workflow instantly when a new lead submits the Tally form and extracts their data. |
| Find Affiliate Sheet | Google Drive | Searches target Drive folder for an affiliate-specific spreadsheet | Extract Form Fields | Sheet Exists? | ## Affiliate Sheet Management<br>Searches for an existing affiliate spreadsheet in Google Drive, branching to create and organize a new one if missing. |
| Sheet Exists? | If | Branches depending on whether a matching sheet file was found | Find Affiliate Sheet | Append Data (Existing Sheet), Create Affiliate Sheet | ## Affiliate Sheet Management<br>Searches for an existing affiliate spreadsheet in Google Drive, branching to create and organize a new one if missing. |
| Create Affiliate Sheet | Google Sheets | Creates a new spreadsheet for a new affiliate code | Sheet Exists? | Move to Target Folder | ## Affiliate Sheet Management<br>Searches for an existing affiliate spreadsheet in Google Drive, branching to create and organize a new one if missing. |
| Move to Target Folder | Google Drive | Moves the newly created spreadsheet into the designated folder | Create Affiliate Sheet | Format Row Data | ## Affiliate Sheet Management<br>Searches for an existing affiliate spreadsheet in Google Drive, branching to create and organize a new one if missing. |
| Format Row Data | Code | Creates a row object matching the expected sheet columns | Move to Target Folder | Append Data (New Sheet) | ## Record Lead Data<br>Appends the captured lead information as a new row in the appropriate Google Sheet. |
| Append Data (New Sheet) | Google Sheets | Appends lead data to the newly created spreadsheet | Format Row Data | Build Slack Message | ## Record Lead Data<br>Appends the captured lead information as a new row in the appropriate Google Sheet. |
| Append Data (Existing Sheet) | Google Sheets | Appends lead data to an already existing affiliate spreadsheet | Sheet Exists? | Build Slack Message | ## Record Lead Data<br>Appends the captured lead information as a new row in the appropriate Google Sheet. |
| Build Slack Message | Code | Builds a Slack Block Kit notification payload | Append Data (Existing Sheet), Append Data (New Sheet) | Send Lead Notification | ## Team Notification<br>Constructs a formatted message summarizing the lead and delivers it to a Slack channel. |
| Send Lead Notification | Slack | Posts the notification into Slack | Build Slack Message |  | ## Team Notification<br>Constructs a formatted message summarizing the lead and delivers it to a Slack channel.<br>The Slack app must be added/invited to the target channel to successfully post messages. |
| Sticky Note6 | Sticky Note | Workspace documentation and setup guidance |  |  |  |
| Sticky Note | Sticky Note | Visual annotation for form capture section |  |  |  |
| Sticky Note1 | Sticky Note | Visual annotation for sheet management section |  |  |  |
| Sticky Note2 | Sticky Note | Visual annotation for lead recording section |  |  |  |
| Sticky Note3 | Sticky Note | Visual annotation for Slack notification section |  |  |  |
| Sticky Note4 | Sticky Note | Visual warning about Slack channel membership |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Tally Trigger node**
   - Node type: **Tally Trigger**
   - Name it: `Submission Trigger`
   - Configure:
     - **Form ID**: `ODajpY`
   - Credentials:
     - Connect a **Tally API** credential
   - This will be the workflow entry point.

3. **Add a Set node**
   - Node type: **Set**
   - Name it: `Extract Form Fields`
   - Add the following fields:
     - `Name` as string:  
       `{{ $json.question_W54VPJ.trim().replace(/\s+/g, ' ').replace(/\b\w/g, c => c.toUpperCase()) }}`
     - `Email Address` as string:  
       `{{ $json.question_aYWq09.toLowerCase() }}`
     - `Phone Number` as string:  
       `{{ $json.question_62qERe }}`
     - `Affiliation Code` as string:  
       `{{ $json.question_72QroL.replace(/\s+/g, '').toLowerCase() }}`
     - `Submission Date` as string:  
       `{{ $json.createdAt.toDateTime().format('MM/dd/yyyy') }}`
     - `Submission Time` as string:  
       `{{ $json.createdAt.toDateTime().format('HH:mm') }}`
   - Connect **Submission Trigger -> Extract Form Fields**

4. **Add a Google Drive node to search for an existing affiliate sheet**
   - Node type: **Google Drive**
   - Name it: `Find Affiliate Sheet`
   - Resource: **File/Folder**
   - Search target: **files**
   - Drive: **My Drive**
   - Folder: set the folder ID to your affiliate submissions folder  
     Example from this workflow: `1H5x2r0sV0xJGbaZXW-lDjZTUmiZtefv0`
   - Query string:  
     `{{ $json["Affiliation Code"] + "_Submissions" }}`
   - Return all: **true**
   - Fields to return: `name`, `id`
   - Enable **Always Output Data**
   - Credentials:
     - Connect **Google Drive OAuth2**
   - Connect **Extract Form Fields -> Find Affiliate Sheet**

5. **Add an If node for existence checking**
   - Node type: **If**
   - Name it: `Sheet Exists?`
   - Configure a condition:
     - Left value: `{{ $json.keys() }}`
     - Operation: **contains**
     - Right value: `name`
   - True branch means the file exists.
   - False branch means no file was found.
   - Connect **Find Affiliate Sheet -> Sheet Exists?**

6. **Add the existing-sheet append node**
   - Node type: **Google Sheets**
   - Name it: `Append Data (Existing Sheet)`
   - Operation: **Append**
   - Document ID:  
     `{{ $json.id }}`
   - Sheet Name:  
     `{{ $json.name }}`
   - Use **Define Below** mapping
   - Map:
     - `Name` -> `{{ $('Extract Form Fields').item.json.Name }}`
     - `Email Address` -> `{{ $('Extract Form Fields').item.json["Email Address"] }}`
     - `Phone Number` -> `{{ $('Extract Form Fields').item.json["Phone Number"] }}`
     - `Submission Date` -> `{{ $('Extract Form Fields').item.json["Submission Date"] }}`
     - `Submission Time` -> `{{ $('Extract Form Fields').item.json["Submission Time"] }}`
   - Credentials:
     - Connect **Google Sheets OAuth2**
   - Connect **Sheet Exists? true -> Append Data (Existing Sheet)**

7. **Add the spreadsheet creation node for the false branch**
   - Node type: **Google Sheets**
   - Name it: `Create Affiliate Sheet`
   - Resource: **Spreadsheet**
   - Title:  
     `{{ $('Extract Form Fields').item.json["Affiliation Code"] + "_Submissions" }}`
   - Add one initial sheet/tab with title:  
     `{{ $('Extract Form Fields').item.json["Affiliation Code"] + "_Submissions" }}`
   - Credentials:
     - Connect **Google Sheets OAuth2**
   - Connect **Sheet Exists? false -> Create Affiliate Sheet**

8. **Add a Google Drive move node**
   - Node type: **Google Drive**
   - Name it: `Move to Target Folder`
   - Operation: **Move**
   - File ID:  
     `{{ $json.spreadsheetId }}`
   - Drive: **My Drive**
   - Destination folder: your affiliate submissions folder  
     Example: `1H5x2r0sV0xJGbaZXW-lDjZTUmiZtefv0`
   - Credentials:
     - Connect **Google Drive OAuth2**
   - Connect **Create Affiliate Sheet -> Move to Target Folder**

9. **Add a Code node to shape the row for a new sheet**
   - Node type: **Code**
   - Name it: `Format Row Data`
   - Language: JavaScript
   - Paste:
     ```javascript
     return [{
       json: {
         "Name": $('Extract Form Fields').first().json.Name,
         "Email Address": $('Extract Form Fields').first().json["Email Address"],
         "Phone Number": $('Extract Form Fields').first().json["Phone Number"],
         "Submission Date": $('Extract Form Fields').first().json["Submission Date"],
         "Submission Time": $('Extract Form Fields').first().json["Submission Time"]
       }
     }];
     ```
   - Connect **Move to Target Folder -> Format Row Data**

10. **Add the append node for the newly created sheet**
    - Node type: **Google Sheets**
    - Name it: `Append Data (New Sheet)`
    - Operation: **Append**
    - Document ID:  
      `{{ $('Create Affiliate Sheet').item.json.spreadsheetId }}`
    - Sheet Name/Sheet selector by ID:  
      `{{ $('Create Affiliate Sheet').item.json.sheets[0].properties.sheetId }}`
    - Mapping mode: **Auto-map input data**
    - Ensure the target schema contains these columns:
      - Name
      - Email Address
      - Phone Number
      - Submission Date
      - Submission Time
    - Options:
      - Cell format: `RAW`
    - Credentials:
      - Connect **Google Sheets OAuth2**
    - Connect **Format Row Data -> Append Data (New Sheet)**

11. **Add a Code node to build the Slack payload**
    - Node type: **Code**
    - Name it: `Build Slack Message`
    - Language: JavaScript
    - Paste:
      ```javascript
      const blocks = [
        {
          "type": "header",
          "text": {
              "type": "plain_text",
              "text": "🎉 New Lead Received!",
              "emoji": true
          }
        },
        {
          "type": "section",
          "text": {
              "type": "mrkdwn",
              "text": `*👤 Name:* ${ $input.first().json.Name }`
          }
        },
        {
          "type": "section",
          "text": {
              "type": "mrkdwn",
              "text": `*📧 Email Address:* ${ $input.first().json["Email Address"] }`
          }
        }
      ];

      const phone_number = $input.first().json["Phone Number"];

      if ( phone_number !== "" ) {
         blocks.push({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": `*📞 Phone Number:* ${ phone_number }`
            }
          }
        ); 
      }

      blocks.push({
          "type": "section",
          "text": {
              "type": "mrkdwn",
              "text": `_Affiliated from *${ $('Extract Form Fields').first().json["Affiliation Code"] }*_`
          }
        }
      );

      return {
        "json": {
            "blocks": blocks
        }
      };
      ```
    - Connect both:
      - **Append Data (Existing Sheet) -> Build Slack Message**
      - **Append Data (New Sheet) -> Build Slack Message**

12. **Add a Slack node**
    - Node type: **Slack**
    - Name it: `Send Lead Notification`
    - Resource/operation equivalent: send message
    - Message type: **Block**
    - Channel selection mode: by ID
    - Channel ID: `C0A2JFJ3L3Z` or your target channel
    - Blocks input:  
      `{{ $json }}`
    - Leave plain text minimal if desired; this workflow uses `=`
    - Credentials:
      - Connect a **Slack API** credential using a bot token
      - Required permission typically includes `chat:write`
    - Important:
      - The Slack app must be invited to the target channel
    - Connect **Build Slack Message -> Send Lead Notification**

13. **Verify the connection layout**
    - `Submission Trigger -> Extract Form Fields -> Find Affiliate Sheet -> Sheet Exists?`
    - True branch: `Sheet Exists? -> Append Data (Existing Sheet) -> Build Slack Message -> Send Lead Notification`
    - False branch:  
      `Sheet Exists? -> Create Affiliate Sheet -> Move to Target Folder -> Format Row Data -> Append Data (New Sheet) -> Build Slack Message -> Send Lead Notification`

14. **Configure credentials**
    - **Tally API**
      - Connect your Tally account/API key
      - Ensure the chosen form is accessible
    - **Google Drive OAuth2**
      - Must allow listing files, reading file metadata, and moving files
    - **Google Sheets OAuth2**
      - Must allow spreadsheet creation and row appends
    - **Slack API**
      - Use a bot token such as `xoxb-...`
      - Ensure the app has `chat:write`
      - Invite the bot to the channel

15. **Prepare external prerequisites**
    - Create a Google Drive folder for affiliate spreadsheets
    - Copy its Folder ID and use it in both Drive nodes
    - Ensure your Tally form includes fields corresponding to:
      - name
      - email
      - phone
      - affiliation code
    - If Tally question IDs differ, replace:
      - `question_W54VPJ`
      - `question_aYWq09`
      - `question_62qERe`
      - `question_72QroL`

16. **Test both branches**
    - First, submit a form with a new affiliation code to test sheet creation
    - Then, submit another form with the same code to test existing-sheet append behavior

17. **Recommended hardening improvements after reproduction**
    - Add validation for missing Tally fields before normalization
    - Add a stricter file-type check after Drive search
    - Handle duplicate Drive search results explicitly
    - Add fallback text to the Slack message
    - Consider creating sheet headers explicitly if your Google Sheets append behavior requires them

### Sub-workflow setup
This workflow does **not** invoke or depend on any sub-workflow nodes. There are no Execute Workflow or workflow-call dependencies to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How it works: 1. Form Trigger: Listens for new public submissions via a Tally.so webhook. 2. Data Routing: Extracts the affiliation code and searches Google Drive for an existing matching spreadsheet. 3. Sheet Management: Automatically appends the lead to the correct affiliate sheet. If no sheet exists, it creates a new one and moves it to the target Drive folder. 4. Notification: Formats the lead details and sends an instant alert to a specified Slack channel. | Workspace documentation |
| Setup Process: Deploy the n8n environment using Docker Compose. Connect your Tally.so API Key. Create a Google Drive folder (e.g., `Affiliate_Submissions`) and copy its Folder ID. Set up Google OAuth credentials and connect the Google Drive and Google Sheets nodes. Connect a Slack Bot Token (`xoxb-...`) with `chat:write` scope. Update node parameters: Add the Tally form, Drive Folder IDs, and Slack Channel ID. | Workspace documentation |
| Customization: Modify your Tally form to capture extra info like budget or company size and map the new fields in n8n. Change the target Google Drive directory for generated affiliate sheets. Adjust the Slack message template to @mention specific sales team members or include custom formatting. | Workspace documentation |
| The Slack app must be added/invited to the target channel to successfully post messages. | Slack posting requirement |