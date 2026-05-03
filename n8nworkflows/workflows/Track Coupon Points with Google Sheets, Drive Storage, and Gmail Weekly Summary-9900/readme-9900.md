Track Coupon Points with Google Sheets, Drive Storage, and Gmail Weekly Summary

https://n8nworkflows.xyz/workflows/track-coupon-points-with-google-sheets--drive-storage--and-gmail-weekly-summary-9900


# Track Coupon Points with Google Sheets, Drive Storage, and Gmail Weekly Summary

### 1. Workflow Overview

This workflow is designed to track loyalty coupon points by integrating Google Sheets, Google Drive, and Gmail with n8n automation. It enables users to submit coupon details via a form, upload proof screenshots to Google Drive, store and organize all coupon data in Google Sheets, and receive weekly email summaries of claimable points based on calculated eligibility dates.

Logical blocks:

- **1.1 Input Reception and Data Storage:** Receives coupon data and screenshots via a web form trigger, uploads screenshots to Google Drive, calculates claimable date, cleans and formats data, then appends it as a new row in a Google Sheet.
- **1.2 Weekly Claimable Points Summary:** Triggered weekly by a schedule node, retrieves coupon data from Google Sheets, filters coupons that have reached their claimable date, generates an HTML summary table, and sends the summary via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Storage

**Overview:**  
Handles the user input of coupon details and screenshot uploads, processes the data by calculating when points become claimable, and stores the information systematically in Google Sheets and Google Drive.

**Nodes Involved:**  
- Tracking Form  
- Upload file  
- calcDate  
- Merge  
- Clean Set  
- Append row in sheet  

**Node Details:**

- **Tracking Form**  
  - *Type:* Form Trigger  
  - *Role:* Entry point to capture coupon details and screenshot files via a web form.  
  - *Configuration:*  
    - Form path: "CouponTracker"  
    - Fields include Coupon Name (text), Account Used (text), Program (dropdown with options Payback, Miles and More, OTHER), Base Points Expected (number), Extra Points Expected (number), Purchase Date (date), Redeem Delay (number), Screenshot (single file upload).  
    - Displays confirmation text after submission.  
  - *Connections:* Outputs to "Upload file" node.  
  - *Edge cases:* Form submission errors, invalid field values.  
  - *Version:* 1

- **Upload file**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the screenshot file received via the form to a designated Google Drive folder.  
  - *Configuration:*  
    - File name is dynamically set from the uploaded Screenshot filename.  
    - Drive and Folder IDs are required (must be configured).  
    - Input data taken from the Screenshot field of the form submission.  
  - *Connections:* Outputs uploaded file metadata to "Merge".  
  - *Edge cases:* Authentication failure, invalid folder/drive ID, upload errors, file size limits.  
  - *Version:* 3

- **calcDate**  
  - *Type:* Code (JavaScript)  
  - *Role:* Calculates the "Claimable Date" by adding the redeem delay (days) to the purchase date.  
  - *Configuration:*  
    - Uses $json properties "Purchase Date" and "Redeem Delay" to compute the claimable date.  
    - Outputs the claimable date as ISO string (YYYY-MM-DD).  
  - *Connections:* Outputs to "Merge".  
  - *Edge cases:* Invalid date formats, missing fields, non-integer redeem delay.  
  - *Version:* 2

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines the outputs from "calcDate" (coupon data + calculated date) and "Upload file" (uploaded file info) into one item.  
  - *Configuration:* Combine mode with "combineAll" option.  
  - *Connections:* Outputs combined data to "Clean Set".  
  - *Edge cases:* Mismatched input arrays, empty inputs.  
  - *Version:* 3.2

- **Clean Set**  
  - *Type:* Set  
  - *Role:* Normalizes and renames fields to align with Google Sheets schema, including linking the uploaded file’s webViewLink as "Coupon Screen".  
  - *Configuration:*  
    - Maps fields:  
      - Coupon Name  
      - Account Used to Buy (from "Account Used")  
      - Program  
      - Points (Base Points Expected)  
      - Bonus Points (Extra Points Expected)  
      - Purchase Date  
      - Redeem Delay (days)  
      - Claimable Date (from calcDate)  
      - Coupon Screen (webViewLink from uploaded file)  
  - *Connections:* Outputs to "Append row in sheet".  
  - *Edge cases:* Missing fields, mapping errors.  
  - *Version:* 2

- **Append row in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Inserts a new row into the specified Google Sheet with the cleaned and formatted coupon data.  
  - *Configuration:*  
    - Document ID and Sheet Name must be set to target the appropriate Google Sheet.  
    - Uses automatic mapping based on field names matching the sheet’s schema.  
  - *Connections:* End of this block.  
  - *Edge cases:* Authentication failure, invalid sheet ID, quota limits, schema mismatch.  
  - *Version:* 4.6

---

#### 1.2 Weekly Claimable Points Summary

**Overview:**  
Runs weekly to fetch all coupon rows from the Google Sheet, filter those which are now claimable based on the current date, generate an HTML email summary table, and send it via Gmail.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- filteredRows  
- rewrite  
- Send a message  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the summary workflow every Saturday at 10:00 AM weekly.  
  - *Configuration:*  
    - Interval set to weekly, trigger day Saturday (6), trigger hour 10.  
  - *Connections:* Outputs to "Get row(s) in sheet".  
  - *Edge cases:* Scheduler misconfiguration, timezone issues.  
  - *Version:* 1.2

- **Get row(s) in sheet**  
  - *Type:* Google Sheets  
  - *Role:* Reads all rows from the configured Google Sheet where coupon data is stored.  
  - *Configuration:*  
    - Requires Document ID and Sheet Name configured to point to the correct data sheet.  
  - *Connections:* Outputs rows to "filteredRows".  
  - *Edge cases:* Authentication errors, empty sheet, quota limits.  
  - *Version:* 4.6

- **filteredRows**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters the rows to keep only those coupons whose "Claimable Date" is today or earlier. Adds current date string to output.  
  - *Configuration:*  
    - Uses current date and compares it with "Claimable Date" field of each row.  
  - *Connections:* Outputs filtered items to "rewrite".  
  - *Edge cases:* Invalid date formats in sheet, empty input arrays.  
  - *Version:* 2

- **rewrite**  
  - *Type:* Code (JavaScript)  
  - *Role:* Constructs an HTML email body summarizing claimable points with a table. If no claimable points, returns a message stating none are available this week.  
  - *Configuration:*  
    - Builds an HTML table with columns: Row, Coupon Name, Account Used to Buy, Program, Points, Bonus Points, Purchase Date, Redeem Delay (days), Claimable Date, Coupon Screen.  
    - Styles the table for readability.  
  - *Connections:* Outputs email body and checked date to "Send a message".  
  - *Edge cases:* Empty filtered list, malformed data entries.  
  - *Version:* 2

- **Send a message**  
  - *Type:* Gmail  
  - *Role:* Sends an email containing the weekly summary of claimable points to a configured recipient.  
  - *Configuration:*  
    - Recipient email must be set in "sendTo".  
    - Subject includes the date checked dynamically.  
    - Email body includes the generated HTML content.  
  - *Connections:* End of this block.  
  - *Edge cases:* Authentication failure, email quota limits, invalid recipient address.  
  - *Version:* 2.1

---

### 3. Summary Table

| Node Name         | Node Type          | Functional Role                           | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                          |
|-------------------|--------------------|-----------------------------------------|-----------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Tracking Form     | Form Trigger       | Receive coupon data and screenshots     | -                           | Upload file             | ## Saving Coupon Data and Screenshots: stores coupon details and screenshots in Drive and Sheets.  |
| Upload file       | Google Drive       | Upload screenshot file to Drive         | Tracking Form               | Merge                   | ## Saving Coupon Data and Screenshots                                                            |
| calcDate          | Code               | Calculate claimable date from purchase  | Tracking Form               | Merge                   | ## Saving Coupon Data and Screenshots                                                            |
| Merge             | Merge              | Combine file upload and coupon data     | Upload file, calcDate       | Clean Set                | ## Saving Coupon Data and Screenshots                                                            |
| Clean Set         | Set                | Format data fields for Sheets insertion | Merge                      | Append row in sheet      | ## Saving Coupon Data and Screenshots                                                            |
| Append row in sheet| Google Sheets      | Append coupon data as new row            | Clean Set                  | -                       | ## Saving Coupon Data and Screenshots                                                            |
| Schedule Trigger  | Schedule Trigger   | Weekly trigger to start summary          | -                           | Get row(s) in sheet     | ## Weekly Claimable Points Check: weekly check and summary email of claimable points               |
| Get row(s) in sheet| Google Sheets      | Retrieve all coupon rows                  | Schedule Trigger            | filteredRows             | ## Weekly Claimable Points Check                                                                 |
| filteredRows      | Code               | Filter coupons with claimable date ≤ today | Get row(s) in sheet      | rewrite                  | ## Weekly Claimable Points Check                                                                 |
| rewrite           | Code               | Generate HTML summary email body         | filteredRows                | Send a message           | ## Weekly Claimable Points Check                                                                 |
| Send a message    | Gmail              | Send summary email to user                | rewrite                     | -                       | ## Weekly Claimable Points Check                                                                 |
| Sticky Note       | Sticky Note        | Documentation overview                    | -                           | -                       | ## Coupon Points Tracker overview and instructions                                               |
| Sticky Note1      | Sticky Note        | Explanation of data and screenshot saving| -                           | -                       | ## Saving Coupon Data and Screenshots                                                            |
| Sticky Note2      | Sticky Note        | Explanation of weekly summary email       | -                           | -                       | ## Weekly Claimable Points Check                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "Tracking Form"  
   - Type: Form Trigger  
   - Path: "CouponTracker"  
   - Form title: "Coupon Points Tracker"  
   - Add fields:  
     - Coupon Name (text, required)  
     - Account Used (text, required)  
     - Program (dropdown: Payback, Miles and More, OTHER; required)  
     - Base Points Expected (number, required)  
     - Extra Points Expected (number)  
     - Purchase Date (date, required)  
     - Redeem Delay (number, required)  
     - Screenshot (file upload, single file)  
   - Set form submitted text: "Your response has been recorded"

2. **Create a Google Drive node** named "Upload file"  
   - Operation: Upload file  
   - File Name: expression `{{$json["Screenshot"]["filename"]}}`  
   - Drive ID: Set your Google Drive ID  
   - Folder ID: Set your target folder ID in Drive  
   - Input Data Field Name: "Screenshot"  
   - Connect output of "Tracking Form" to input of "Upload file"

3. **Create a Code node** named "calcDate"  
   - Language: JavaScript  
   - Code:
   ```javascript
   const purchaseDate = new Date($json['Purchase Date']);
   const delay = parseInt($json['Redeem Delay']);
   const claimableDate = new Date(purchaseDate);
   claimableDate.setDate(purchaseDate.getDate() + delay);
   return [{
     json: {
       ...$json,
       'Claimable Date': claimableDate.toISOString().split('T')[0]
     }
   }];
   ```
   - Connect output of "Tracking Form" to input of "calcDate"

4. **Create a Merge node** named "Merge"  
   - Mode: Combine  
   - Combine By: Combine All  
   - Connect outputs:  
     - "Upload file" → input 2 of "Merge"  
     - "calcDate" → input 1 of "Merge"

5. **Create a Set node** named "Clean Set"  
   - Keep Only Set: true  
   - Set the following fields with expressions:
     - Coupon Name: `{{$json["Coupon Name"]}}`  
     - Account Used to Buy: `{{$json["Account Used"]}}`  
     - Program: `{{$json["Program"]}}`  
     - Points: `{{$json["Base Points Expected"]}}`  
     - Bonus Points: `{{$json["Extra Points Expected"]}}`  
     - Purchase Date: `{{$json["Purchase Date"]}}`  
     - Redeem Delay (days): `{{$json["Redeem Delay"]}}`  
     - Claimable Date: `{{$json["Claimable Date"]}}`  
     - Coupon Screen: `{{$json["webViewLink"]}}` (from uploaded file metadata)  
   - Connect output of "Merge" to "Clean Set"

6. **Create a Google Sheets node** named "Append row in sheet"  
   - Operation: Append  
   - Document ID: your Google Sheet ID  
   - Sheet Name: your sheet name or gid  
   - Mapping Mode: Auto map input data  
   - Connect output of "Clean Set" to "Append row in sheet"

---

7. **Create a Schedule Trigger node** named "Schedule Trigger"  
   - Trigger interval: Weekly  
   - Trigger at day: Saturday (6)  
   - Trigger at hour: 10:00  
   - Connect output to "Get row(s) in sheet"

8. **Create a Google Sheets node** named "Get row(s) in sheet"  
   - Operation: Read rows (default)  
   - Document ID: same Google Sheet ID as above  
   - Sheet Name: same as above  
   - Connect output to "filteredRows"

9. **Create a Code node** named "filteredRows"  
   - Language: JavaScript  
   - Code:
   ```javascript
   const today = new Date();
   const todayStr = today.toISOString().slice(0, 10);
   const filtered = items.filter(item => {
     const claimDate = new Date(item.json["Claimable Date"]);
     return claimDate <= today;
   }).map(item => {
     return {
       json: {
         ...item.json,
         currentDate: todayStr
       }
     };
   });
   return filtered;
   ```
   - Connect output to "rewrite"

10. **Create a Code node** named "rewrite"  
    - Language: JavaScript  
    - Code:
    ```javascript
    const today = new Date();
    const todayStr = today.toISOString().slice(0, 10);
    const rows = items
      .map(i => i.json)
      .filter(r => new Date(r["Claimable Date"]) <= today);

    if (rows.length === 0) {
      return [{
        json: {
          body: "<p>No claimable points this week.</p>",
          dateChecked: todayStr
        }
      }];
    }

    let html = `<p>Claimable Bonus Points as of ${todayStr}:</p>`;
    html += `<table border="1" cellpadding="4" cellspacing="0" style="border-collapse: collapse; font-family: Arial, sans-serif; font-size: 14px;">`;
    html += "<thead><tr>";

    const headers = [
      "Row",
      "Coupon Name",
      "Account Used to Buy",
      "Program",
      "Points",
      "Bonus Points",
      "Purchase Date",
      "Redeem Delay (days)",
      "Claimable Date",
      "Coupon Screen"
    ];

    headers.forEach(h => {
      html += `<th style="background:#eee;">${h}</th>`;
    });
    html += "</tr></thead><tbody>";

    rows.forEach(r => {
      html += "<tr>";
      html += `<td>${r.row_number || ""}</td>`;
      html += `<td>${r["Coupon Name"] || ""}</td>`;
      html += `<td>${r["Account Used to Buy"] || ""}</td>`;
      html += `<td>${r["Program"] || ""}</td>`;
      html += `<td>${r["Points"] || ""}</td>`;
      html += `<td>${r["Bonus Points"] || ""}</td>`;
      html += `<td>${r["Purchase Date"] || ""}</td>`;
      html += `<td>${r["Redeem Delay (days)"] || ""}</td>`;
      html += `<td>${r["Claimable Date"] || ""}</td>`;
      html += `<td>${r["Coupon Screen"] || ""}</td>`;
      html += "</tr>";
    });

    html += "</tbody></table>";

    return [{
      json: {
        body: html,
        dateChecked: todayStr
      }
    }];
    ```
    - Connect output to "Send a message"

11. **Create a Gmail node** named "Send a message"  
    - Operation: Send Email  
    - Send To: set recipient email address  
    - Subject: `Weekly Claimable Bonus Points - Checked {{$json.dateChecked}}`  
    - Message: `{{$json.body}}` (HTML content)  
    - Optional: Disable attribution append  
    - Connect output of "rewrite" node to this node

---

12. **Credential Setup:**  
- Google Drive credentials with rights to upload files to the target folder.  
- Google Sheets credentials with rights to read and append rows to the target document.  
- Gmail credentials authorized to send emails on your behalf.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow helps track loyalty points and claimable bonuses automatically, ensuring no points are missed or forgotten.                                                                                                  | Workflow purpose                                                                                   |
| Video overview and setup instructions can be found at: https://n8n.io/blog/automate-loyalty-points-tracking-with-google-sheets-and-gmail                                                                                 | External blog link                                                                                |
| Requires Google OAuth credentials setup in n8n for Drive, Sheets, and Gmail services.                                                                                                                                       | Credential setup                                                                                   |
| Adjust the schedule trigger timing as needed to fit your preferred summary frequency.                                                                                                                                       | Customization advice                                                                              |
| Store screenshots in a dedicated Drive folder for easy retrieval and backup.                                                                                                                                                 | Best practice advice                                                                              |
| Use consistent field names in the Google Sheet matching the workflow's mapping to prevent data mismatch.                                                                                                                    | Implementation note                                                                              |
| Ensure date formats are consistent (ISO 8601) in Google Sheets to avoid filter errors.                                                                                                                                       | Data consistency tip                                                                             |

---

*Disclaimer: The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.*