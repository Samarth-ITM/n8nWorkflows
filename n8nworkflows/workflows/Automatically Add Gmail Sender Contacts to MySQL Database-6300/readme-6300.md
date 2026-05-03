Automatically Add Gmail Sender Contacts to MySQL Database

https://n8nworkflows.xyz/workflows/automatically-add-gmail-sender-contacts-to-mysql-database-6300


# Automatically Add Gmail Sender Contacts to MySQL Database

### 1. Workflow Overview

This workflow automatically processes incoming Gmail emails to extract the sender's name and email address, then adds or updates this contact information into a MySQL database. It is targeted mainly at sales, marketing, or any teams needing to build and maintain a client contact list without manual data entry. The workflow logically breaks down into the following blocks:

- **1.1 Input Reception:** Listen continuously for new emails arriving in Gmail.
- **1.2 Data Extraction:** Parse the sender's full name and email address from the received email.
- **1.3 Database Upsert:** Insert a new record or update an existing one in a MySQL contacts table based on the sender's email.
- **1.4 Guidance & Configuration Notes:** Provides setup instructions and customization hints for users.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new incoming emails in Gmail with a polling interval of one minute. It triggers the workflow execution for each new email detected.

- **Nodes Involved:**  
  - Receive Email

- **Node Details:**  
  - **Receive Email**  
    - Type: Gmail Trigger  
    - Role: Polls Gmail account for new emails every minute, triggering the workflow on new messages.  
    - Configuration: Polling interval set to every minute, no filters applied by default (can be customized).  
    - Inputs: None (trigger node).  
    - Outputs: Emits new email data to downstream nodes.  
    - Credentials: Requires Gmail OAuth2 credentials.  
    - Potential Failures: Authentication failures (expired or invalid OAuth token), Gmail API rate limits, network timeouts.  
    - Notes: Sticky note nearby suggests adding filters like labels for selective email processing.

#### 1.2 Data Extraction

- **Overview:**  
  Extracts the sender’s name and email address from the 'From' header of each received email. If no name is present, returns null for the name field.

- **Nodes Involved:**  
  - Extract Client Name and Email

- **Node Details:**  
  - **Extract Client Name and Email**  
    - Type: Code Node (JavaScript)  
    - Role: Parses the 'From' field of the email, separating name and email.  
    - Configuration: Runs once per input item.  
    - Key Expression:  
      ```js
      let email = $json.From.trim();
      let name = null;
      if (email.includes('<')) {
        name = email.split('<')[0].trim();
        email = email.split('<')[1].replace('>', '').trim();
      }
      return { "name": name, "email": email };
      ```
    - Inputs: Email data from Gmail trigger node.  
    - Outputs: Object with `name` (nullable string) and `email` (string).  
    - Edge Cases: If the 'From' field is malformed or empty, output may be incorrect or null; no explicit error handling is implemented.  
    - Version: Uses n8n v2 code node syntax.  
    - Notes: Relies on the standard email format "Name <email@domain.com>". If format varies, extraction might fail.

#### 1.3 Database Upsert

- **Overview:**  
  Inserts a new contact or updates the existing contact’s name in the MySQL database using the email as the unique key.

- **Nodes Involved:**  
  - Insert New Client in MySQL

- **Node Details:**  
  - **Insert New Client in MySQL**  
    - Type: MySQL Node  
    - Role: Performs an UPSERT operation on the contacts table to store sender info.  
    - Configuration:  
      - Table: `contacts`  
      - Operation: `upsert`  
      - Match Field: `email` column matched against incoming `email` value  
      - Data to Send: `name` column updated with extracted `name` value (nullable)  
      - On Error: Continues workflow even if errors occur (e.g., duplicate key, connection issues).  
    - Inputs: Output from code node with `name` and `email`.  
    - Outputs: Always outputs data to next node or end (though none connected).  
    - Credentials: MySQL credentials configured with "BT Operations Database".  
    - Edge Cases:  
      - Duplicate emails handled by upsert, but if database constraints or connection errors occur, insert/update may fail.  
      - If the `name` is null, updates the name column to null (allowed by schema).  
    - Notes: Sticky note advises users to ensure the table has `name` (nullable) and `email` (unique) columns properly configured.

#### 1.4 Guidance & Configuration Notes

- **Overview:**  
  Provides users with detailed instructions on prerequisites, setup, and customization options for the workflow.

- **Nodes Involved:**  
  - Sticky Note (multiple instances)

- **Node Details:**  
  - **Sticky Note (Main Overview)**  
    - Contains use cases, setup requirements (Gmail and MySQL credentials, database table schema), and a high-level workflow description.  
  - **Sticky Note1 (MySQL Setup Instructions)**  
    - Advises creating a MySQL table with `name` (nullable) and `email` (unique) columns before use.  
  - **Sticky Note2 (Customization Tips for Gmail Trigger)**  
    - Suggests adding filters such as Gmail labels to restrict which emails are processed.  
  - **Sticky Note3 (Customizing MySQL Node)**  
    - Explains how to extend the data saved to MySQL by adding more fields from the Gmail data (e.g., subject line, message ID).  
  - Notes: These sticky notes do not affect workflow execution but are essential for correct operation and user adaptation.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                         | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                                                                                                                                 |
|---------------------------|---------------------|---------------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note               | Sticky Note         | Workflow overview and instructions    | None                     | None                       | ## This workflow processes emails received in Gmail and adds the sender's **name** and **email address** to a MySQL database. Use cases and setup instructions included.                                                    |
| Sticky Note1              | Sticky Note         | MySQL database setup instructions     | None                     | None                       | ## Please setup this node first - Create a table in your MySQL database with 'name' (nullable) and 'email' (unique) columns.                                                                                            |
| Sticky Note2              | Sticky Note         | Gmail Trigger customization tips      | None                     | None                       | ## Customize this to your liking - You can add filters to this node to only save certain contact emails. Example: Label filter.                                                                                           |
| Sticky Note3              | Sticky Note         | MySQL node customization guidance     | None                     | None                       | ## Customizing this Workflow - Save additional Gmail fields like Subject, MessageID, ThreadID, Snippet, Recipient Info to MySQL by configuring the MySQL node.                                                             |
| Receive Email             | Gmail Trigger       | Trigger on new Gmail emails            | None                     | Extract Client Name and Email |                                                                                                                                                                                                                             |
| Extract Client Name and Email | Code Node        | Parse sender's name and email          | Receive Email            | Insert New Client in MySQL  |                                                                                                                                                                                                                             |
| Insert New Client in MySQL | MySQL Node         | Upsert contact info into MySQL table   | Extract Client Name and Email | None                   | ## Please setup this node first - Create a table in your MySQL database with 'name' (nullable) and 'email' (unique) columns.                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Gmail Trigger Node**  
   - Add a **Gmail Trigger** node named `Receive Email`.  
   - Set the polling interval to **every minute**.  
   - Do not apply any filters initially (optional: add filters like labels to limit emails processed).  
   - Configure with valid **Gmail OAuth2 credentials**.

2. **Add a Code Node to Extract Name and Email**  
   - Add a **Code** node named `Extract Client Name and Email`.  
   - Set mode to **Run Once For Each Item**.  
   - Use the following JavaScript code:  
     ```js
     let email = $json.From.trim();
     let name = null;
     if (email.includes('<')) {
       name = email.split('<')[0].trim();
       email = email.split('<')[1].replace('>', '').trim();
     }
     return { "name": name, "email": email };
     ```  
   - Connect the output of `Receive Email` to the input of this node.

3. **Add a MySQL Node for Upsert Operation**  
   - Add a **MySQL** node named `Insert New Client in MySQL`.  
   - Set operation to **Upsert**.  
   - Select or enter the target table name (default: `contacts`).  
   - Set the **Column to Match On** as `email`. Use `={{ $json.email }}` expression for the value.  
   - Define the data to send: map the `name` column to `={{ $json.name }}`.  
   - Configure with valid **MySQL credentials** connected to your contacts database.  
   - Set error handling to **Continue on Error** to avoid workflow termination on DB errors.  
   - Connect the output of `Extract Client Name and Email` to this node.

4. **Optional: Add Sticky Notes**  
   - Insert **Sticky Note** nodes to provide user instructions and customization hints as per the workflow overview.  
   - Text content should include use cases, setup instructions for Gmail and MySQL credentials, database schema requirements, and customization ideas for filters and additional data fields.

5. **Finalize and Activate Workflow**  
   - Review connections: `Receive Email` → `Extract Client Name and Email` → `Insert New Client in MySQL`.  
   - Test the workflow by sending emails to the Gmail account and verifying inserts/updates in the MySQL contacts table.  
   - Adjust filters or add additional fields as needed to customize.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is designed for automatic client contact collection via Gmail and MySQL integration.                               | Workflow purpose and use cases.                                                                          |
| Ensure your MySQL table schema includes a `name` column allowing NULL values and an `email` column with UNIQUE constraint.      | Database schema requirement.                                                                             |
| Gmail OAuth2 credential and MySQL credential must be preconfigured in n8n credentials manager.                                    | Credential setup prerequisites.                                                                          |
| You can add Gmail label filters in the Gmail Trigger node to process only specific emails.                                        | Customizing input reception.                                                                              |
| Consider expanding the data saved to MySQL by adding fields such as Subject, MessageID, ThreadID, Snippet, and Recipient Info.   | Workflow customization suggestions.                                                                      |
| For troubleshooting, check for authentication errors, API limits, and database connectivity issues.                              | Common failure points and debugging hints.                                                               |

---

**Disclaimer:** The text provided exclusively derives from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.