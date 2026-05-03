Sync New Subscribers from Google Sheets to MailerLite without Duplicates

https://n8nworkflows.xyz/workflows/sync-new-subscribers-from-google-sheets-to-mailerlite-without-duplicates-7681


# Sync New Subscribers from Google Sheets to MailerLite without Duplicates

### 1. Workflow Overview

This workflow automates the synchronization of new subscribers from a Google Sheet to MailerLite while preventing duplicates. It is designed for marketing and CRM teams who maintain subscriber data in Google Sheets and want to ensure that only new contacts are added to MailerLite, avoiding redundant entries.

The workflow is logically divided into four main blocks:

- **1.1 Manual Trigger:** Initiates the workflow on demand to maintain control over execution.
- **1.2 Data Retrieval from Google Sheets:** Fetches subscriber rows from a specified Google Sheet.
- **1.3 Subscriber Existence Check in MailerLite:** Queries MailerLite to determine if a subscriber already exists.
- **1.4 Subscriber Creation and Workflow Termination:** Creates new subscribers in MailerLite if not found and ends the workflow for existing ones.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block allows the user to start the workflow manually, providing control over execution timing, useful for testing and scheduled runs.

- **Nodes Involved:**  
  - Start Workflow  
  - Sticky Note (explaining the manual trigger purpose)

- **Node Details:**

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow on demand.  
    - Configuration: No parameters; triggers workflow manually.  
    - Input: None  
    - Output: Connects to "Get row(s) in sheet" node.  
    - Edge Cases: None specifically; manual start reduces unexpected runs.  
    - Notes: Enables controlled execution.

  - **Sticky Note (bc39a624-695d-48c7-a295-e40c3a7b84fa)**  
    - Content explains the importance of manual triggering for control, testing, and execution.

#### 1.2 Data Retrieval from Google Sheets

- **Overview:**  
  Retrieves subscriber data rows from a specific Google Sheet, serving as the data source for the synchronization process.

- **Nodes Involved:**  
  - Get row(s) in sheet  
  - Sticky Note (explaining the Google Sheets data retrieval)

- **Node Details:**

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Fetches rows from the configured spreadsheet and sheet.  
    - Configuration:  
      - Operation: Get Rows  
      - Document ID and Sheet Name set via expression (linked to user-provided Google Sheets document and sheet).  
      - Credentials: Google Sheets OAuth2 with necessary scopes.  
    - Input: Triggered from Manual Trigger.  
    - Output: Passes data rows downstream.  
    - Edge Cases:  
      - Authentication errors if OAuth token expired or invalid.  
      - Empty or malformed sheets causing no data or errors.  
      - Rate limits on Google API.  
    - Notes: Critical data source node.

  - **Sticky Note (cbce1376-b59f-4a7d-a6a6-ac410ae1ca39)**  
    - Describes the purpose of fetching rows from Google Sheets and its importance.

#### 1.3 Subscriber Existence Check in MailerLite

- **Overview:**  
  Checks if each subscriber email from Google Sheets already exists in MailerLite to prevent duplicate entries.

- **Nodes Involved:**  
  - Get a subscriber  
  - Sticky Note (explaining subscriber check and branching)

- **Node Details:**

  - **Get a subscriber**  
    - Type: MailerLite node  
    - Role: Queries MailerLite API to get subscriber details by email.  
    - Configuration:  
      - Operation: Get subscriber  
      - Subscriber ID: Extracted dynamically from item JSON field `Email`.  
      - Credentials: MailerLite API key.  
      - On Error: Continue workflow on error (important for handling "subscriber not found" scenarios).  
      - Always outputs data regardless of success or failure.  
    - Input: Receives rows from Google Sheets node.  
    - Output: Two branches:  
      - Success (subscriber exists)  
      - Error (subscriber not found, treated as new subscriber)  
    - Edge Cases:  
      - API authentication failures.  
      - API rate limits or timeouts.  
      - Incorrect email format causing errors.  
      - Handling subscribers not found gracefully via error branch.  
    - Notes: Central decision node for branching workflow.

  - **Sticky Note (3a331aef-f9c8-4368-984f-6221cd1a4a5b)**  
    - Explains the purpose of this node: differentiating existing and new subscribers, enabling targeted processing.

#### 1.4 Subscriber Creation and Workflow Termination

- **Overview:**  
  For existing subscribers, the workflow ends gracefully to avoid duplication. For new subscribers, it creates the subscriber in MailerLite and assigns them to the appropriate group.

- **Nodes Involved:**  
  - End workflow (No Operation node)  
  - Create subscriber and assign to group (HTTP Request node)  
  - Sticky Notes (explaining both end workflow and create subscriber steps)

- **Node Details:**

  - **End workflow**  
    - Type: No Operation (NoOp) node  
    - Role: Terminates workflow cleanly for existing subscribers.  
    - Configuration: No parameters.  
    - Input: Success branch from "Get a subscriber" node (subscriber found).  
    - Output: None (ends workflow).  
    - Edge Cases: None; safe termination point.  
    - Notes: Prevents duplication and keeps logic clean.

  - **Create subscriber and assign to group**  
    - Type: HTTP Request  
    - Role: Creates a new subscriber in MailerLite and assigns them to a group.  
    - Configuration:  
      - Method: POST  
      - URL: https://connect.mailerlite.com/api/subscribers  
      - Body: JSON, composed dynamically with subscriber fields:  
        - email, first_name, last_name, company, country, group_id (all from Google Sheets row data)  
      - Authentication: MailerLite API predefined credential (API Key).  
    - Input: Error branch from "Get a subscriber" node (subscriber not found).  
    - Output: None (workflow ends after creation).  
    - Edge Cases:  
      - API authentication failure.  
      - API rate limits.  
      - Invalid or missing data in JSON causing request failure.  
      - Network timeouts or errors.  
    - Notes: Crucial for adding new contacts and maintaining list segmentation.

  - **Sticky Note (7a52a415-bd9d-4b39-85d6-d3cb8e0b4b8a)**  
    - Explains the importance of ending workflow cleanly for existing users.

  - **Sticky Note (55e7ee44-8deb-4c24-ba01-2cf8dc065acb)**  
    - Explains the creation of new subscribers and group assignment.

---

### 3. Summary Table

| Node Name                      | Node Type          | Functional Role                           | Input Node(s)             | Output Node(s)                                | Sticky Note                                                                                              |
|--------------------------------|--------------------|-----------------------------------------|---------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Start Workflow                 | Manual Trigger     | Initiates workflow on demand             | None                      | Get row(s) in sheet                           | Step 1: Manual Trigger 🖐️⚡ Allows controlled execution and testing.                                   |
| Get row(s) in sheet            | Google Sheets      | Fetches subscriber data rows             | Start Workflow            | Get a subscriber                              | Step 2: Get Rows from Google Sheets 📊📥 Retrieves latest data for processing.                          |
| Get a subscriber               | MailerLite         | Checks if subscriber exists in MailerLite| Get row(s) in sheet       | End workflow (success branch)<br>Create subscriber and assign to group (error branch) | Step 3: Get Subscriber from MailerLite 📧🔍 Differentiates existing vs new subscribers.                 |
| End workflow                  | No Operation       | Ends workflow for existing subscribers  | Get a subscriber (success)| None                                          | Step 4: End Workflow for Existing Users 🛑✅ Prevents duplicates and maintains workflow clarity.        |
| Create subscriber and assign to group | HTTP Request      | Creates new subscriber and assigns group| Get a subscriber (error)  | None                                          | Step 4: Create Subscriber & Assign to Group 🆕👥 Adds new subscribers and segments them properly.       |
| Sticky Note                   | Sticky Note        | Explanatory notes                        | N/A                       | N/A                                           | See individual sticky notes for detailed comments on each step.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed.  
   - Position it as the workflow start node.

2. **Add Google Sheets Node "Get row(s) in sheet"**  
   - Operation: Get Rows  
   - Document ID: Use your Google Sheets document ID (e.g., `your_spreadsheet_id`)  
   - Sheet Name: Use sheet identifier (e.g., `gid=0`) or sheet name.  
   - Credentials: Connect your Google Sheets OAuth2 credentials with access to Google Sheets and Drive scopes.  
   - Connect output of Manual Trigger to this node.

3. **Add MailerLite Node "Get a subscriber"**  
   - Operation: Get subscriber  
   - Subscriber ID: Set to expression `{{$json["Email"]}}` to use the email from each row.  
   - Credentials: Use your MailerLite API key credentials.  
   - Set "On Error" to "Continue" to handle missing subscribers gracefully.  
   - Connect output of Google Sheets node to this node.

4. **Add No Operation Node "End workflow"**  
   - No parameters.  
   - Connect the **success** output of "Get a subscriber" node (subscriber found) to this node.  
   - This node ends the workflow for existing subscribers.

5. **Add HTTP Request Node "Create subscriber and assign to group"**  
   - Method: POST  
   - URL: `https://connect.mailerlite.com/api/subscribers`  
   - Authentication: Use MailerLite API key credentials.  
   - Body Content-Type: JSON  
   - Request Body JSON (use expression):  
     ```json
     {
       "email": "{{ $json.Email }}",
       "fields": {
         "name": "{{ $json.first_name }}",
         "last_name": "{{ $json.last_name }}",
         "company": "{{ $json.Company }}",
         "country": "{{ $json.Country }}"
       },
       "groups": ["{{ $json.group_id }}"]
     }
     ```
   - Connect the **error** output of "Get a subscriber" node (subscriber not found) to this node.

6. **Save the workflow and activate it.**

7. **Additional Setup Notes:**  
   - Ensure your Google Sheet has columns: `Email`, `first_name`, `last_name`, `Company`, `Country`, `group_id`.  
   - Confirm that all credentials (Google OAuth2 and MailerLite API) are properly configured and tested.  
   - Test the workflow by running the manual trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Before running this workflow, ensure Google credentials have access to both Google Sheets and Drive APIs, and your MailerLite API key is valid and authorized for subscriber operations. Your Google Sheet must include the header columns: Email, first_name, last_name, Company, Country, and group_id to map subscriber fields correctly.                                                                                                                                                                           | Workflow Prerequisites Sticky Note                                                                        |
| For detailed MailerLite API documentation and to manage groups or update subscriber fields, visit: https://developers.mailerlite.com/reference#create-subscriber                                                                                                                                                                                                                                                                                                                                                   | MailerLite API Reference                                                                                   |
| The workflow uses error handling on the MailerLite "Get a subscriber" node to branch between existing and new subscribers. This design ensures no duplicates and handles API errors gracefully without stopping the workflow.                                                                                                                                                                                                                                                                                         | Workflow design best practice                                                                              |
| This workflow is ideal for users managing subscriber lists in Google Sheets and wanting to keep MailerLite subscriber lists synchronized, segmented, and free of duplicates via on-demand or scheduled runs.                                                                                                                                                                                                                                                                                                         | Use case description                                                                                       |

---

This document provides a complete, structured understanding and instructions to implement and maintain the "Sync New Subscribers from Google Sheets to MailerLite without Duplicates" workflow in n8n.