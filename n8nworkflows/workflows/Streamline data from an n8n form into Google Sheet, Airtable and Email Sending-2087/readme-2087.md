Streamline data from an n8n form into Google Sheet, Airtable and Email Sending

https://n8nworkflows.xyz/workflows/streamline-data-from-an-n8n-form-into-google-sheet--airtable-and-email-sending-2087


# Streamline data from an n8n form into Google Sheet, Airtable and Email Sending

---

### 1. Workflow Overview

This workflow is designed to streamline data collection from an n8n form and efficiently distribute the captured data into Google Sheets and Airtable, while also sending personalized emails to the form submitter. It targets use cases where user input data needs to be simultaneously logged in multiple data repositories and followed up with email communication.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures user-submitted form data including Name, City, and Email.
- **1.2 Data Processing:** Extracts and formats date and time information from the form submission metadata and standardizes all collected fields.
- **1.3 Data Storage:** Appends the formatted data to Google Sheets and creates a new record in Airtable.
- **1.4 Email Dispatch:** Sends two customized emails to the submitter using Gmail nodes with slightly different templates and subject lines.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon form submission. It collects the user's Name, City, and Email via form fields, initiating subsequent data processing and handling.

- **Nodes Involved:**  
  - n8n Form Trigger

- **Node Details:**

  - **n8n Form Trigger**  
    - *Type:* Trigger node, initiates workflow on form submission.  
    - *Configuration:*  
      - Path: Unique webhook path `c07c8eb6-cf56-4941-91cc-e3cb31c90b5c`.  
      - Form Title: "Data Colleacation" (note: slight typo in "Collection").  
      - Form Fields:  
        - "What's your name ?" (required)  
        - "Where do you live ?" (required)  
        - "Your Email ?" (required)  
    - *Expressions:* None; directly captures form data.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Passes collected form data and metadata including `submittedAt`.  
    - *Potential Failures:*  
      - Webhook availability or authorization issues.  
      - Missing required fields (though enforced by form).  
      - Network connectivity issues.  
    - *Version Requirements:* n8n version supporting formTrigger v2 (generally recent).  

#### 2.2 Data Processing

- **Overview:**  
  Extracts date and time from the submission timestamp and formats all relevant fields for uniformity. This ensures data consistency before storage and emailing.

- **Nodes Involved:**  
  - Extracting Date and Time Fields from 'submittedAt' Field (Code node)  
  - Format the Fields (Set node)

- **Node Details:**

  - **Extracting Date and Time Fields from 'submittedAt' Field**  
    - *Type:* Code node (JavaScript execution).  
    - *Configuration:*  
      - Processes all input items.  
      - Extracts `submittedAt` date-time string, splits to separate `Date` (YYYY-MM-DD) and `Time` (HH:mm:ss).  
      - Removes original `submittedAt` field.  
      - Adds new fields: `Date` and `Time` to JSON.  
    - *Key Expressions:*  
      - Uses JavaScript Date object and ISO string manipulation.  
    - *Inputs:* From n8n Form Trigger node.  
    - *Outputs:* JSON with extracted `Date` and `Time`, no `submittedAt`.  
    - *Potential Failures:*  
      - Malformed or missing `submittedAt` field.  
      - Timezone issues if `submittedAt` is not in ISO format.  
      - Code execution errors if input structure differs.  
    - *Version Requirements:* n8n supporting Code node v2.

  - **Format the Fields**  
    - *Type:* Set node (data transformation).  
    - *Configuration:*  
      - Sets fields explicitly:  
        - `Name` from form field "What's your name ?"  
        - `City` from "Where do you live ?"  
        - `Date` from extracted `Date` field  
        - `Time` from extracted `Time` field  
        - `Email` from "Your Email ?"  
      - Ensures string values for all fields.  
    - *Key Expressions:* Uses expression syntax: `={{ $json['fieldName'] }}`.  
    - *Inputs:* From Code node.  
    - *Outputs:* Standardized JSON with five fields.  
    - *Potential Failures:*  
      - Missing form fields due to unexpected input format.  
      - Expression evaluation errors if fields are undefined.  
    - *Version Requirements:* Set node v3.2.

#### 2.3 Data Storage

- **Overview:**  
  Stores the formatted data simultaneously into two external data repositories: Google Sheets and Airtable.

- **Nodes Involved:**  
  - Google Sheets  
  - Airtable

- **Node Details:**

  - **Google Sheets**  
    - *Type:* Google Sheets node (API integration).  
    - *Configuration:*  
      - Operation: Append new rows.  
      - Document ID: Linked to shared Google Sheet template.  
      - Sheet Name: Uses GID=0 (default sheet).  
      - Columns mapped: `Name`, `City`, `Email`, `Date`, `Time`.  
    - *Expressions:* Fields mapped with `={{ $json.FieldName }}` expressions.  
    - *Inputs:* From Format the Fields node.  
    - *Outputs:* Passes data unchanged to next node.  
    - *Credentials:* OAuth2 credentials for Google Sheets API.  
    - *Potential Failures:*  
      - Authentication failure in Google API.  
      - Sheet or document ID mismatch or access issues.  
      - API rate limits or network errors.  
    - *Version Requirements:* Google Sheets node v4.2 or later.

  - **Airtable**  
    - *Type:* Airtable node (API integration).  
    - *Configuration:*  
      - Operation: Create new record.  
      - Base ID and Table ID correspond to shared Airtable template.  
      - Columns: `Name`, `City`, `Email`, `Date`, `Time`.  
      - Mapping mode: Explicit column definition.  
    - *Expressions:* Uses `={{ $json.FieldName }}` for each column.  
    - *Inputs:* From Format the Fields node.  
    - *Outputs:* Passes data to Gmail node.  
    - *Credentials:* Airtable Personal Access Token.  
    - *Potential Failures:*  
      - Authentication errors with Airtable API.  
      - Incorrect Base or Table IDs.  
      - API rate limits or network issues.  
    - *Version Requirements:* Airtable node v2.

#### 2.4 Email Dispatch

- **Overview:**  
  Sends two distinct emails to the submitter’s email address. The first email is triggered after storing data in Airtable, and the second after data is appended to Google Sheets, each with personalized content.

- **Nodes Involved:**  
  - Gmail  
  - Gmail1

- **Node Details:**

  - **Gmail**  
    - *Type:* Gmail node (email sending).  
    - *Configuration:*  
      - Recipient: `={{ $json.fields.Email }}` (from Airtable output).  
      - Subject: "Testing Text Message Delivery".  
      - Message: Text email with personalized greeting using `{{ $json.fields.Name }}`.  
      - Email Type: Plain text.  
    - *Inputs:* From Airtable node.  
    - *Outputs:* None (terminal).  
    - *Credentials:* Gmail OAuth2 credentials.  
    - *Potential Failures:*  
      - Gmail OAuth authentication errors.  
      - Invalid email addresses.  
      - Network or API rate limits.  
    - *Version Requirements:* Gmail node v2.1.

  - **Gmail1**  
    - *Type:* Gmail node (email sending).  
    - *Configuration:*  
      - Recipient: `={{ $json.Email }}` (from Google Sheets output).  
      - Subject: "Testing Text Message Delivery , ( {{ $json.Date }} )" — Date included in subject.  
      - Message: Similar personalized text as Gmail node.  
      - Email Type: Plain text.  
    - *Inputs:* From Google Sheets node.  
    - *Outputs:* None (terminal).  
    - *Credentials:* Same Gmail OAuth2 credentials as Gmail node.  
    - *Potential Failures:* Similar to Gmail node.  
    - *Version Requirements:* Gmail node v2.1.

---

### 3. Summary Table

| Node Name                                               | Node Type                  | Functional Role                      | Input Node(s)                                  | Output Node(s)                         | Sticky Note                                                                                              |
|---------------------------------------------------------|----------------------------|------------------------------------|-----------------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------|
| n8n Form Trigger                                        | Form Trigger               | Captures form submission data      | None                                          | Extracting Date and Time Fields        |                                                                                                       |
| Extracting Date and Time Fields from 'submittedAt' Field | Code                       | Extracts Date and Time from timestamp | n8n Form Trigger                              | Format the Fields                     |                                                                                                       |
| Format the Fields                                       | Set                        | Standardizes data fields            | Extracting Date and Time Fields               | Google Sheets, Airtable               |                                                                                                       |
| Google Sheets                                          | Google Sheets              | Appends data to Google Sheet        | Format the Fields                             | Gmail1                               |                                                                                                       |
| Airtable                                               | Airtable                   | Creates new record in Airtable      | Format the Fields                             | Gmail                                |                                                                                                       |
| Gmail                                                  | Gmail                      | Sends personalized email (template 1) | Airtable                                    | None                                 |                                                                                                       |
| Gmail1                                                 | Gmail                      | Sends personalized email (template 2) | Google Sheets                               | None                                 |                                                                                                       |
| Sticky Note                                            | Sticky Note                | Workflow description note           | None                                          | None                                 | Contains detailed Workflow Description                                                                |
| Sticky Note1                                           | Sticky Note                | Node documentation links            | None                                          | None                                 | Links to official node documentation for all nodes                                                    |
| Sticky Note2                                           | Sticky Note                | Empty (placeholder)                  | None                                          | None                                 |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add **n8n Form Trigger** node.  
   - Configure webhook path (e.g., `c07c8eb6-cf56-4941-91cc-e3cb31c90b5c`).  
   - Set form title: "Data Colleacation".  
   - Add form fields (all required):  
     - "What's your name ?"  
     - "Where do you live ?"  
     - "Your Email ?"  

2. **Add a Code Node to Extract Date and Time:**  
   - Add **Code** node.  
   - Use this JavaScript code to process input items:  
     ```js
     for (const item of $input.all()) {
       const submittedAt = new Date(item.json['submittedAt']);
       const date = submittedAt.toISOString().split('T')[0];
       const time = submittedAt.toISOString().split('T')[1].split('.')[0];
       delete item.json['submittedAt'];
       item.json['Date'] = date;
       item.json['Time'] = time;
     }
     return $input.all();
     ```  
   - Connect output of Form Trigger to this Code node.

3. **Add a Set Node to Format Fields:**  
   - Add **Set** node.  
   - Configure fields to set:  
     - Name = `={{ $json["What's your name ?"] }}`  
     - City = `={{ $json["Where do you live ?"] }}`  
     - Date = `={{ $json.Date }}`  
     - Time = `={{ $json.Time }}`  
     - Email = `={{ $json["Your Email ?"] }}`  
   - Connect Code node output to this Set node.

4. **Add Google Sheets Node to Append Data:**  
   - Add **Google Sheets** node.  
   - Operation: Append.  
   - Document ID: Use your Google Sheet document ID (e.g., from template link).  
   - Sheet Name: Use sheet GID=0 or appropriate sheet name.  
   - Map columns:  
     - Name, City, Email, Date, Time (all string fields).  
   - Set up **Google Sheets OAuth2** credentials.  
   - Connect Set node output to Google Sheets node.

5. **Add Airtable Node to Create Record:**  
   - Add **Airtable** node.  
   - Operation: Create.  
   - Connect your Airtable Base ID and Table ID (from your Airtable template).  
   - Map fields: Name, City, Email, Date, Time as strings.  
   - Set up **Airtable Personal Access Token** credentials.  
   - Connect Set node output to Airtable node.

6. **Add Gmail Node to Send First Email:**  
   - Add **Gmail** node.  
   - Configure:  
     - Send To: `={{ $json.fields.Email }}` (from Airtable output).  
     - Subject: "Testing Text Message Delivery".  
     - Message (plain text):  
       ```
       Dear {{ $json.fields.Name }} ..

       Hey there! Just testing to see if this message goes through. Let me know if you receive it. 

       Thanks! 
       Support Team
       ```  
   - Set up **Gmail OAuth2** credentials.  
   - Connect Airtable node output to Gmail node.

7. **Add Second Gmail Node to Send Another Email:**  
   - Add another **Gmail** node (rename as Gmail1).  
   - Configure:  
     - Send To: `={{ $json.Email }}` (from Google Sheets output).  
     - Subject: `=Testing Text Message Delivery , ( {{ $json.Date }} )`.  
     - Message (plain text):  
       ```
       Dear {{ $json.Name }} ..

       Hey there! Just testing to see if this message goes through. Let me know if you receive it. 

       Thanks! 
       Support Team
       ```  
   - Use same Gmail OAuth2 credentials as first Gmail node.  
   - Connect Google Sheets node output to Gmail1 node.

8. **Connect All Nodes as Follows:**  
   - n8n Form Trigger → Code node (Extract Date and Time)  
   - Code node → Set node (Format Fields)  
   - Set node → Google Sheets node  
   - Set node → Airtable node  
   - Airtable node → Gmail node  
   - Google Sheets node → Gmail1 node  

9. **Test the Workflow:**  
   - Activate the workflow.  
   - Submit test data via the form endpoint.  
   - Confirm data appends to Google Sheets and Airtable.  
   - Verify that emails arrive with correct personalized content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow process video explaining the workflow visually.                                                                           | https://www.canva.com/design/DAF8PpnKJ0Q/2egm6F7B9H0Vm_8KFrghPw/watch?utm_content=DAF8PpnKJ0Q&utm_campaign=designshare&utm_medium=link&utm_source=editor |
| Google Sheet template for data appending.                                                                                        | https://docs.google.com/spreadsheets/d/1Ss6AEwaXpAl54YQAQDf1z6SRyh6pj719-A9eOzf2Dv4/edit?usp=sharing                                         |
| Airtable template for record creation.                                                                                           | https://airtable.com/appIIeJ18fnPkNyNS/shrhlIxwdsDF6Fy0S/tblZvKuOMmtHnv5TH/viwgKQKnV5gosIcCe                                                  |
| Official n8n documentation for nodes used in this workflow:                                                                       | 1. https://docs.n8n.io/nodes/n8n-nodes-base.formTrigger  
2. https://docs.n8n.io/nodes/n8n-nodes-base.code  
3. https://docs.n8n.io/nodes/n8n-nodes-base.set  
4. https://docs.n8n.io/nodes/n8n-nodes-base.airtable  
5. https://docs.n8n.io/nodes/n8n-nodes-base.googleSheets  
6. https://docs.n8n.io/nodes/n8n-nodes-base.gmail                                             |
| Ensure OAuth2 credentials for Google Sheets and Gmail are properly configured with sufficient API scopes.                         | Credential setup is critical for successful data writing and email sending.                                                                  |
| Confirm Airtable Personal Access Token has write permissions to the Base and Table used.                                          | Prevents unauthorized errors during record creation.                                                                                        |
| Validate email addresses to avoid rejections or bounces.                                                                         | Use form validation and error handling if needed.                                                                                           |

---

This document provides a complete and precise reference for understanding, reproducing, and maintaining the described n8n workflow, including technical details, node configurations, and integration considerations.