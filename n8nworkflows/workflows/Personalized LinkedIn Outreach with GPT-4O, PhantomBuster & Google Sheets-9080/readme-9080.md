Personalized LinkedIn Outreach with GPT-4O, PhantomBuster & Google Sheets

https://n8nworkflows.xyz/workflows/personalized-linkedin-outreach-with-gpt-4o--phantombuster---google-sheets-9080


# Personalized LinkedIn Outreach with GPT-4O, PhantomBuster & Google Sheets

### 1. Workflow Overview

This workflow automates a personalized LinkedIn outreach campaign leveraging GPT-4O for AI-generated icebreaker messages, PhantomBuster for LinkedIn automation, and Google Sheets for data management. It targets marketing or sales professionals seeking to efficiently scale connection requests with highly customized messaging. The workflow is structured into four logical blocks:

- **1.1 Automated Scheduling:** Triggers the workflow twice daily at preset business hours to ensure consistent outreach volume.
- **1.2 Data Processing & AI Personalization:** Reads LinkedIn prospect data from Google Sheets, limits batch size, and uses GPT-4O to create personalized icebreaker messages.
- **1.3 Lead Management & Campaign Execution:** Saves enriched prospect data back to Google Sheets and triggers PhantomBuster to execute LinkedIn connection requests.
- **1.4 Cleanup & Notification:** Cleans up processed data from sheets to avoid duplicates and sends an email notification upon campaign completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Automated Scheduling

- **Overview:**  
This block schedules the workflow runs twice per day at 2 AM and 7 PM server time, initiating the outreach process automatically without manual input.

- **Nodes Involved:**  
  - Schedule Trigger1  
  - Schedule Trigger2  
  - Delete rows or columns from sheet1  

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Fires workflow at 2 AM daily  
    - Configuration: Interval trigger set to hour 2  
    - Inputs: None (start node)  
    - Outputs: Connected to "Delete rows or columns from sheet1"  
    - Edge cases: Timezone differences may affect execution time; ensure server timezone matches intended schedule.

  - **Schedule Trigger2**  
    - Type: Schedule Trigger  
    - Role: Fires workflow at 7 PM daily  
    - Configuration: Interval trigger set to hour 19  
    - Inputs: None (start node)  
    - Outputs: Connected to "Delete rows or columns from sheet1"  
    - Edge cases: Same as Schedule Trigger1 regarding timezone.

  - **Delete rows or columns from sheet1**  
    - Type: Google Sheets node  
    - Role: Clears first 10 rows in the input sheet before processing new data  
    - Configuration:  
      - Operation: Delete  
      - Sheet ID and name configured dynamically (referencing Google Sheets OAuth2 credentials)  
      - Number to delete: 10 rows  
    - Inputs: From both Schedule Triggers  
    - Outputs: Connected to "Get row(s) in sheet"  
    - Edge cases: If sheet is empty or has fewer than 10 rows, may cause errors or no action. Ensure sheet structure is consistent.

---

#### 2.2 Data Processing & AI Personalization

- **Overview:**  
Reads LinkedIn prospect data, limits the batch size to 10 leads, generates personalized icebreaker messages using GPT-4O, and prepares data for campaign execution.

- **Nodes Involved:**  
  - Get row(s) in sheet  
  - Limit1  
  - Personalize Outreach  
  - Sticky Note (explaining this block)

- **Node Details:**

  - **Get row(s) in sheet**  
    - Type: Google Sheets node  
    - Role: Retrieves LinkedIn prospect data from the input spreadsheet  
    - Configuration:  
      - Reads all rows from the sheet (gid=0)  
      - Uses Google Sheets OAuth2 credentials  
    - Inputs: From "Delete rows or columns from sheet1"  
    - Outputs: Connected to "Limit1"  
    - Edge cases: Empty or malformed spreadsheet data may cause downstream errors. Sheet permissions must be correctly set.

  - **Limit1**  
    - Type: Limit node  
    - Role: Restricts processing to 10 leads per run to control API usage and costs  
    - Configuration: maxItems set to 10  
    - Inputs: From "Get row(s) in sheet"  
    - Outputs: Connected to "Personalize Outreach"  
    - Edge cases: If fewer than 10 rows are present, processes all available rows.

  - **Personalize Outreach**  
    - Type: OpenAI (Langchain) node  
    - Role: Generates short, punchy icebreaker messages personalized per LinkedIn profile using GPT-4O  
    - Configuration:  
      - Model: GPT-4O  
      - System prompt: Defines assistant as an intelligent writing helper  
      - User prompt: Inputs LinkedIn profile fields (firstName, lastName, location, title, companyName, etc.) formatted as JSON string  
      - Output: JSON containing a single key "icebreaker" with generated text  
    - Inputs: From "Limit1"  
    - Outputs: Connected to "Add to Google Sheet"  
    - Credentials: Requires OpenAI API key  
    - Edge cases: API rate limits, network timeouts, or malformed input data may cause failures. The prompt expects specific JSON format; parsing errors must be handled.

---

#### 2.3 Lead Management & Campaign Execution

- **Overview:**  
Appends processed lead data with icebreakers to an output Google Sheet, aggregates all processed data, and triggers a PhantomBuster agent to perform LinkedIn connection requests.

- **Nodes Involved:**  
  - Add to Google Sheet  
  - Aggregate  
  - Trigger PhantomBuster Agent  
  - Sticky Note (explaining this block)

- **Node Details:**

  - **Add to Google Sheet**  
    - Type: Google Sheets node  
    - Role: Saves processed leads and personalized icebreakers to a destination spreadsheet  
    - Configuration:  
      - Operation: Append  
      - Columns mapped include id, name, title, first_name, last_name, linkedin_url, icebreaker, photo_url  
      - Uses Google Sheets OAuth2 credentials  
    - Inputs: From "Personalize Outreach"  
    - Outputs: Connected to "Aggregate"  
    - Edge cases: Incorrect mapping or missing sheet permissions may cause append failures.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Collects all appended data items into a single aggregated dataset  
    - Configuration: Default aggregate all item data  
    - Inputs: From "Add to Google Sheet"  
    - Outputs: Connected to "Trigger PhantomBuster Agent"  
    - Edge cases: Data aggregation errors if input data is inconsistent.

  - **Trigger PhantomBuster Agent**  
    - Type: HTTP Request node  
    - Role: Calls PhantomBuster API to launch the LinkedIn connection request campaign  
    - Configuration:  
      - Method: POST  
      - URL: PhantomBuster agent launch endpoint  
      - Headers: Includes PhantomBuster API key from credentials  
      - Body: Sends the PhantomBuster agent ID via parameter extracted from a sticky note variable  
    - Inputs: From "Aggregate"  
    - Outputs: Connected to "Delete rows or columns from sheet" (cleanup)  
    - Credentials: PhantomBuster API key required  
    - Edge cases: API authentication failures, invalid agent ID, network issues.

---

#### 2.4 Cleanup & Notification

- **Overview:**  
Removes processed rows from input and output sheets to prevent duplication, then sends a notification email confirming campaign completion.

- **Nodes Involved:**  
  - Delete rows or columns from sheet  
  - Send a message  
  - Sticky Note (explaining this block)

- **Node Details:**

  - **Delete rows or columns from sheet**  
    - Type: Google Sheets node  
    - Role: Deletes 10 rows from the output sheet after campaign execution to maintain data hygiene  
    - Configuration:  
      - Operation: Delete  
      - Sheet and Document IDs configured dynamically  
      - Number to delete: 10  
    - Inputs: From "Trigger PhantomBuster Agent"  
    - Outputs: Connected to "Send a message"  
    - Edge cases: If fewer than 10 rows exist, operation may fail or partially delete.

  - **Send a message**  
    - Type: Gmail node  
    - Role: Sends an email notification indicating successful completion of the LinkedIn connection request campaign  
    - Configuration:  
      - Recipient: Email address configured in node parameters  
      - Subject: "Connection Request Sent"  
      - Message body: Simple plaintext notification  
      - Uses Gmail OAuth2 credentials  
    - Inputs: From "Delete rows or columns from sheet"  
    - Outputs: None (end node)  
    - Edge cases: Email sending issues due to authentication, quota limits, or incorrect recipient address.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                                  | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                                                                         |
|-------------------------------|---------------------------|-------------------------------------------------|---------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1              | Schedule Trigger          | Triggers workflow at 2 AM daily                   | None                            | Delete rows or columns from sheet1  | STEP 1: Automated Scheduling... (runs campaign at 10 AM, 5 PM, hands-free automation)                                |
| Schedule Trigger2              | Schedule Trigger          | Triggers workflow at 7 PM daily                   | None                            | Delete rows or columns from sheet1  | STEP 1: Automated Scheduling...                                                                                      |
| Delete rows or columns from sheet1 | Google Sheets           | Deletes first 10 rows before processing input    | Schedule Trigger1, Schedule Trigger2 | Get row(s) in sheet               |                                                                                                                     |
| Get row(s) in sheet           | Google Sheets             | Reads LinkedIn prospect data from input sheet     | Delete rows or columns from sheet1 | Limit1                            | STEP2: Data Processing & AI Personalization... (reads prospect data, limits leads, generates icebreakers)             |
| Limit1                        | Limit                     | Limits processing to 10 leads per execution       | Get row(s) in sheet             | Personalize Outreach               | STEP2: Data Processing & AI Personalization...                                                                        |
| Personalize Outreach          | OpenAI (Langchain)        | Generates personalized icebreaker messages         | Limit1                         | Add to Google Sheet               | STEP2: Data Processing & AI Personalization...                                                                        |
| Add to Google Sheet          | Google Sheets             | Saves processed leads and icebreakers              | Personalize Outreach            | Aggregate                        | STEP 3: Lead Management & Campaign Execution... (store leads, trigger campaign)                                        |
| Aggregate                    | Aggregate                 | Aggregates all processed data                       | Add to Google Sheet             | Trigger PhantomBuster Agent       | STEP 3: Lead Management & Campaign Execution...                                                                        |
| Trigger PhantomBuster Agent  | HTTP Request              | Launches PhantomBuster agent to send requests      | Aggregate                      | Delete rows or columns from sheet | STEP 3: Lead Management & Campaign Execution...                                                                        |
| Delete rows or columns from sheet | Google Sheets           | Deletes 10 rows from output sheet post campaign    | Trigger PhantomBuster Agent     | Send a message                   | STEP 4: Cleanup & Notification System... (cleanup and notification)                                                   |
| Send a message               | Gmail                     | Sends notification email on campaign completion    | Delete rows or columns from sheet | None                            | STEP 4: Cleanup & Notification System...                                                                               |
| Sticky Note                  | Sticky Note               | Notes for step 2 - AI personalization explanation  | None                           | None                            | STEP2: Data Processing & AI Personalization explanation                                                                |
| Sticky Note1                 | Sticky Note               | Notes for step 1 - Scheduling explanation          | None                           | None                            | STEP 1: Automated Scheduling explanation                                                                               |
| Sticky Note2                 | Sticky Note               | Notes for step 3 - Lead management & campaign exec | None                           | None                            | STEP 3: Lead Management & Campaign Execution explanation                                                               |
| Sticky Note3                 | Sticky Note               | Notes for step 4 - Cleanup & notification           | None                           | None                            | STEP 4: Cleanup & Notification System explanation                                                                      |
| Sticky Note4                 | Sticky Note               | Setup instructions and PhantomBuster Agent ID note | None                           | None                            | ⚙️ SETUP REQUIRED - Configure sheets, OpenAI, PhantomBuster credentials, agent ID, notification email address         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create two Schedule Trigger nodes:**
   - Name: "Schedule Trigger1"  
     - Type: Schedule Trigger  
     - Trigger: Daily at 2:00 AM (hour 2)  
   - Name: "Schedule Trigger2"  
     - Type: Schedule Trigger  
     - Trigger: Daily at 7:00 PM (hour 19)  

2. **Create a Google Sheets node to delete input rows:**
   - Name: "Delete rows or columns from sheet1"  
   - Operation: Delete  
   - Sheet: Input sheet (gid=0)  
   - Document ID: Your input Google Sheet ID  
   - Number to delete: 10  
   - Connect both Schedule Trigger nodes' outputs to this node.

3. **Create a Google Sheets node to read rows:**
   - Name: "Get row(s) in sheet"  
   - Operation: Read rows from input sheet (gid=0)  
   - Document ID: Same input Google Sheet ID  
   - Connect output of "Delete rows or columns from sheet1" to this node.

4. **Create a Limit node:**
   - Name: "Limit1"  
   - Max Items: 10  
   - Connect output of "Get row(s) in sheet" to this node.

5. **Create OpenAI (Langchain) node for personalization:**
   - Name: "Personalize Outreach"  
   - Model: GPT-4O  
   - Credentials: Set your OpenAI API credentials  
   - Messages configuration:  
     - System message: "You are a helpful, intelligent writing assistant."  
     - User prompt: Include LinkedIn profile fields (firstName, lastName, location, title, companyName, titleDescription) formatted as JSON string to generate punchy icebreakers following the provided template.  
   - Output: JSON containing "icebreaker" string  
   - Connect output of "Limit1" to this node.

6. **Create Google Sheets node to append enriched data:**
   - Name: "Add to Google Sheet"  
   - Operation: Append  
   - Sheet: Output sheet (gid=0)  
   - Document ID: Your output Google Sheet ID  
   - Map columns: id, name, title, first_name, last_name, linkedin_url, icebreaker, photo_url  
   - Connect output of "Personalize Outreach" to this node.

7. **Create Aggregate node:**
   - Name: "Aggregate"  
   - Aggregate all item data (default)  
   - Connect output of "Add to Google Sheet" to this node.

8. **Create HTTP Request node to trigger PhantomBuster:**
   - Name: "Trigger PhantomBuster Agent"  
   - Method: POST  
   - URL: https://api.phantombuster.com/api/v2/agents/launch  
   - Headers: X-Phantombuster-Key with your PhantomBuster API key  
   - Body: JSON with parameter "id" set to your PhantomBuster agent ID (can be stored in a variable or sticky note)  
   - Connect output of "Aggregate" to this node.

9. **Create Google Sheets node to delete output rows after campaign:**
   - Name: "Delete rows or columns from sheet"  
   - Operation: Delete  
   - Sheet: Output sheet (gid=0)  
   - Document ID: Output Google Sheet ID  
   - Number to delete: 10  
   - Connect output of "Trigger PhantomBuster Agent" to this node.

10. **Create Gmail node to send notification email:**
    - Name: "Send a message"  
    - Use Gmail OAuth2 credentials  
    - Recipient email: Your notification address  
    - Subject: "Connection Request Sent"  
    - Message: "LinkedIn connection request campaign completed successfully."  
    - Connect output of "Delete rows or columns from sheet" to this node.

11. **Set credentials:**
    - Configure Google Sheets OAuth2 credentials for all Google Sheets nodes.  
    - Add OpenAI API credentials to "Personalize Outreach".  
    - Add PhantomBuster API credentials to "Trigger PhantomBuster Agent".  
    - Add Gmail OAuth2 credentials to "Send a message".

12. **Add Sticky Notes (optional but recommended):**
    - Add notes for each logical step explaining purpose and setup instructions.  
    - Include PhantomBuster agent ID and other setup parameters clearly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| ⚙️ SETUP REQUIRED: Configure Google Sheets credentials, specify source/destination sheets, add OpenAI and PhantomBuster API credentials, enter PhantomBuster Agent ID, and set notification email address before running.           | Sticky Note4 content                                                                                        |
| STEP 1: Automated Scheduling runs the campaign twice daily (10 AM and 5 PM), sending 10 connection requests each time, ensuring hands-free automation without manual intervention.                                                | Sticky Note1 content                                                                                        |
| STEP 2: Data Processing & AI Personalization reads LinkedIn prospect data, limits to 10 leads per run, and uses GPT-4O to create concise, human-like icebreaker messages for personalized outreach.                              | Sticky Note content near Data Processing nodes                                                             |
| STEP 3: Lead Management & Campaign Execution saves processed leads with icebreakers, aggregates data, then triggers PhantomBuster to automate LinkedIn connection requests, coordinating n8n processing with PhantomBuster.     | Sticky Note2 content                                                                                        |
| STEP 4: Cleanup & Notification deletes processed rows from sheets to avoid duplicates and sends a notification email confirming campaign success, maintaining clean data and tracking.                                          | Sticky Note3 content                                                                                        |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow, strictly adhering to content policies with no illegal or protected elements. All data handled is legal and public.