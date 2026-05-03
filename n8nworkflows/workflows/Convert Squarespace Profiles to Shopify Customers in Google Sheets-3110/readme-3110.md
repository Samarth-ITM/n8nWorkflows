Convert Squarespace Profiles to Shopify Customers in Google Sheets

https://n8nworkflows.xyz/workflows/convert-squarespace-profiles-to-shopify-customers-in-google-sheets-3110


# Convert Squarespace Profiles to Shopify Customers in Google Sheets

### 1. Workflow Overview

This workflow automates the conversion of exported Squarespace contact profiles into a Shopify-compatible customer import format using Google Sheets as an intermediary. It is designed for users who export contacts from Squarespace and want to import them into Shopify without manual CSV reformatting.

**Target Use Cases:**  
- Automating the transformation of Squarespace contacts export into Shopify customer import format.  
- Supporting both manual trigger (for on-demand conversions) and webhook trigger (for automated, event-driven imports).  
- Managing data via a shared Google Sheets template with two sheets: one for raw Squarespace profiles and one formatted for Shopify import.

**Logical Blocks:**

- **1.1 Input Reception**  
  Handles data input either via webhook file upload or manual trigger reading from Google Sheets.

- **1.2 Data Extraction & Preparation**  
  Extracts CSV data from webhook file or reads existing Squarespace profiles from Google Sheets.

- **1.3 Data Append to Squarespace Profiles Sheet**  
  Appends or updates the Squarespace Profiles sheet with incoming data to maintain a master list.

- **1.4 Batch Processing Loop**  
  Processes data in batches to handle large datasets efficiently.

- **1.5 Shopify Customers Sheet Update**  
  Converts and writes the processed data into the Shopify Customers sheet in the required format for import.

- **1.6 Workflow Trigger Nodes**  
  Includes manual trigger and webhook nodes to start the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives input data either via a webhook POST request containing a CSV file or via manual trigger for processing existing data in Google Sheets.

**Nodes Involved:**  
- Webhook  
- Manual trigger

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Receives POST requests with CSV file uploads.  
  - Configuration: Path set to `submit-profiles`, allows all origins (`*`), accepts POST requests only.  
  - Inputs: External HTTP POST with multipart/form-data containing a file.  
  - Outputs: Passes binary file data to next node.  
  - Edge Cases: Invalid HTTP methods, missing file in request, large file uploads causing timeouts.

- **Manual trigger**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of workflow for on-demand processing.  
  - Configuration: Default, no parameters.  
  - Inputs: None (manual start).  
  - Outputs: Triggers downstream nodes to read Google Sheets data.  
  - Edge Cases: None significant.

---

#### 1.2 Data Extraction & Preparation

**Overview:**  
Extracts CSV content from the webhook file upload or reads existing Squarespace profiles from Google Sheets.

**Nodes Involved:**  
- Extract items from webhook submission  
- Read Squarespace profiles

**Node Details:**

- **Extract items from webhook submission**  
  - Type: Extract from File  
  - Role: Parses CSV file from webhook binary data into JSON items.  
  - Configuration: Reads from binary property named `file`.  
  - Inputs: Binary file data from Webhook node.  
  - Outputs: JSON array of profile objects.  
  - Edge Cases: Malformed CSV, empty files, encoding issues.

- **Read Squarespace profiles**  
  - Type: Google Sheets  
  - Role: Reads existing Squarespace profiles from the Google Sheets "Squarespace Profiles" sheet.  
  - Configuration: Reads all rows from sheet ID `144532755` in document `1yf_RYZGFHpMyOvD3RKGSvIFY2vumvI4474Qm_1t4-jM`.  
  - Inputs: Trigger from Manual trigger node.  
  - Outputs: JSON array of profile objects.  
  - Credentials: Requires Google Sheets OAuth2 credentials.  
  - Edge Cases: Sheet access errors, empty sheet, API rate limits.

---

#### 1.3 Data Append to Squarespace Profiles Sheet

**Overview:**  
Appends or updates the Squarespace Profiles sheet with incoming or read data to maintain a master list of profiles.

**Nodes Involved:**  
- Append Squarespace profiles

**Node Details:**

- **Append Squarespace profiles**  
  - Type: Google Sheets  
  - Role: Writes incoming profile data into the "Squarespace Profiles" sheet, updating existing entries by matching on Email.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Matching column: Email  
    - Columns mapped: Email, First Name, Last Name, Billing and Shipping details, Order Count, Last Order Date, Subscriber info, etc.  
    - Sheet ID: `144532755` (Squarespace Profiles sheet)  
    - Document ID: `1yf_RYZGFHpMyOvD3RKGSvIFY2vumvI4474Qm_1t4-jM`  
  - Inputs: JSON profile items from data extraction or manual trigger path.  
  - Outputs: Passes updated profiles downstream.  
  - Credentials: Google Sheets OAuth2 required.  
  - Edge Cases: Duplicate emails, partial data updates, API quota limits, schema mismatches.

---

#### 1.4 Batch Processing Loop

**Overview:**  
Splits the profile data into batches of 1000 items to efficiently process large datasets and avoid API limits or timeouts.

**Nodes Involved:**  
- Loop Over Items

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes input data in batches of 1000 items.  
  - Configuration: Batch size set to 1000.  
  - Inputs: JSON array of profiles from Append Squarespace profiles or Extract items from webhook submission.  
  - Outputs: Each batch passed to downstream nodes for processing.  
  - Edge Cases: Last batch smaller than 1000, empty input arrays.

---

#### 1.5 Shopify Customers Sheet Update

**Overview:**  
Transforms profile data into Shopify-compatible customer format and appends or updates the Shopify Customers sheet accordingly.

**Nodes Involved:**  
- Shopify Customers

**Node Details:**

- **Shopify Customers**  
  - Type: Google Sheets  
  - Role: Writes customer data formatted for Shopify import into the "Shopify Customers" sheet.  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Matching column: Email  
    - Columns mapped include: First Name, Last Name, Email, Phone, Address fields, Accepts Email Marketing (always "yes"), Tags (conditionally includes "ground-control" if Last Order Date exists), Company, etc.  
    - Sheet ID: `15798644` (Shopify Customers sheet)  
    - Document ID: `1yf_RYZGFHpMyOvD3RKGSvIFY2vumvI4474Qm_1t4-jM`  
  - Inputs: Batches of profile data from Loop Over Items.  
  - Credentials: Google Sheets OAuth2 required.  
  - Edge Cases: Missing required fields, invalid email formats, API quota limits.

---

#### 1.6 Workflow Trigger Nodes

**Overview:**  
Provide entry points to start the workflow either manually or via webhook.

**Nodes Involved:**  
- Webhook  
- Manual trigger

**Node Details:**  
Covered in section 1.1.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                    | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                           |
|-------------------------------|---------------------|---------------------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook             | Receives CSV file upload via HTTP POST            | -                             | Extract items from webhook submission |                                                                                                                       |
| Extract items from webhook submission | Extract from File   | Parses CSV file from webhook into JSON items      | Webhook                       | Loop Over Items                |                                                                                                                       |
| Manual trigger                | Manual Trigger      | Starts workflow manually                           | -                             | Read Squarespace profiles      |                                                                                                                       |
| Read Squarespace profiles     | Google Sheets       | Reads Squarespace profiles from Google Sheets     | Manual trigger                | Shopify Customers             |                                                                                                                       |
| Append Squarespace profiles   | Google Sheets       | Appends/updates Squarespace Profiles sheet        | Loop Over Items               | Loop Over Items               |                                                                                                                       |
| Loop Over Items               | SplitInBatches      | Processes data in batches of 1000                  | Append Squarespace profiles, Extract items from webhook submission | Shopify Customers, Append Squarespace profiles |                                                                                                                       |
| Shopify Customers             | Google Sheets       | Writes Shopify-compatible customer data           | Loop Over Items               | -                             |                                                                                                                       |
| Sticky Note                  | Sticky Note         | Documentation note                                | -                             | -                             | ## Convert Squarespace profiles\nConvert exported profile from Squarespace to compatible Shopify customers data in csv format\nSample Spreadsheet template\nhttps://docs.google.com/spreadsheets/d/1ZUP7RySMCjQUBAvlZhSE1rOul1FMVHvTSF0QexuV7mQ\n- Squarespace profiles\n- Shopify customers |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Set HTTP Method to POST  
   - Set Path to `submit-profiles`  
   - Allow all origins (`*`)  
   - This node will accept CSV file uploads via HTTP POST.

2. **Create Extract from File Node**  
   - Type: Extract from File  
   - Set Binary Property Name to `file` (matches uploaded file property)  
   - Connect Webhook node output to this node input.

3. **Create SplitInBatches Node**  
   - Type: SplitInBatches  
   - Set Batch Size to 1000  
   - Connect Extract from File node output to this node input.

4. **Create Google Sheets Node for Shopify Customers**  
   - Type: Google Sheets  
   - Operation: appendOrUpdate  
   - Document ID: Use your Google Sheets document ID (from the provided template or your own)  
   - Sheet Name: Shopify Customers sheet (e.g., sheet ID `15798644`)  
   - Matching Columns: Email  
   - Map columns as follows (use expressions referencing incoming JSON fields):  
     - First Name: `={{ $json["First Name"] }}`  
     - Last Name: `={{ $json["Last Name"] }}`  
     - Email: `={{ $json.Email }}`  
     - Phone: `={{ $json["Billing Phone Number"] }}`  
     - Default Address Zip: `={{ $json["Billing Zip"] }}`  
     - Default Address City: `={{ $json["Billing City"] }}`  
     - Default Address Phone: `={{ $json["Billing Phone Number"] }}`  
     - Accepts Email Marketing: `yes` (static)  
     - Default Address Company: `={{ $json["Billing Name"] }}`  
     - Default Address Address1: `={{ $json["Billing Address 1"] }}`  
     - Default Address Address2: `={{ $json["Billing Address 2"] }}`  
     - Default Address Country Code: `={{ $json["Billing Country"] }}`  
     - Default Address Province Code: `={{ $json["Billing Province/State"] }}`  
     - Tags: `=n8n, squarespace, {{ $json["Last Order Date"] ? "ground-control," : "" }}`  
   - Connect SplitInBatches output to this node input.  
   - Configure Google Sheets OAuth2 credentials.

5. **Create Google Sheets Node for Append Squarespace Profiles**  
   - Type: Google Sheets  
   - Operation: appendOrUpdate  
   - Document ID: Same as above  
   - Sheet Name: Squarespace Profiles sheet (e.g., sheet ID `144532755`)  
   - Matching Columns: Email  
   - Map all relevant Squarespace profile fields from input JSON to corresponding columns (Email, First Name, Last Name, Billing and Shipping details, Order Count, Last Order Date, Subscriber info, etc.)  
   - Connect SplitInBatches output to this node input.  
   - Configure Google Sheets OAuth2 credentials.

6. **Connect Append Squarespace Profiles Node output back to SplitInBatches node**  
   - This creates a loop to process batches iteratively.

7. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Connect Manual Trigger output to Google Sheets node "Read Squarespace profiles".

8. **Create Google Sheets Node to Read Squarespace Profiles**  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Document ID: Same as above  
   - Sheet Name: Squarespace Profiles sheet  
   - Connect Manual Trigger output to this node input.

9. **Connect Read Squarespace Profiles output directly to Shopify Customers node**  
   - This allows manual trigger to process existing profiles without webhook upload.

10. **Connect Webhook node output to Extract from File node**  
    - Connect Extract from File node output to SplitInBatches node.

11. **Ensure all Google Sheets nodes use the same OAuth2 credentials**  
    - Set up Google Sheets OAuth2 credentials with appropriate API access.

12. **Test the workflow**  
    - For manual trigger: populate Squarespace Profiles sheet with sample data, run manual trigger, verify Shopify Customers sheet updates.  
    - For webhook: POST a CSV file with Squarespace profiles to the webhook URL, verify data is extracted, appended, and Shopify Customers sheet updated.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Sample Google Sheets template includes two sheets: Squarespace Profiles (input) and Shopify Customers (output).                  | https://docs.google.com/spreadsheets/d/1ZUP7RySMCjQUBAvlZhSE1rOul1FMVHvTSF0QexuV7mQ                      |
| Instructions for exporting Squarespace contacts: Squarespace Dashboard → Contacts → three-dot icon → Export all Contacts.       | Workflow description                                                                                   |
| Instructions for importing customers into Shopify: Shopify Dashboard → Customers → Import customers by CSV.                     | Workflow description                                                                                   |
| Webhook trigger example code snippet provided to POST CSV file to the webhook URL.                                               | Workflow description                                                                                   |
| Workflow creator's other templates available at n8n creators page.                                                              | https://n8n.io/creators/bangank36                                                                      |

---

This documentation fully describes the workflow "Convert Squarespace Profiles to Shopify Customers in Google Sheets," enabling reproduction, modification, and troubleshooting.