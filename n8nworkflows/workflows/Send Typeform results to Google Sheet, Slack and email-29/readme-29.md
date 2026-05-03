Send Typeform results to Google Sheet, Slack and email

https://n8nworkflows.xyz/workflows/send-typeform-results-to-google-sheet--slack-and-email-29


# Send Typeform results to Google Sheet, Slack and email

### 1. Workflow Overview

This workflow automates the processing of new form submissions from a specific Typeform. It captures user-reported problems and routes the data to appropriate destinations based on severity. The key use cases include logging problem reports in a Google Sheet, alerting severe issues via Slack, and notifying less critical problems by email.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures new form submissions from Typeform.
- **1.2 Data Logging:** Appends submission data to a Google Sheet for record-keeping.
- **1.3 Severity Evaluation:** Checks the severity level of the reported problem.
- **1.4 Conditional Notifications:** Sends a Slack message for very severe problems or sends an email notification for less severe issues.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon each new submission of the specified Typeform. It receives form data mapping each question to fields like Name, Email, Severity, and Problem.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**  

  - **Typeform Trigger**  
    - **Type and Role:** Trigger node; listens for new form submissions on Typeform.  
    - **Configuration:** Configured with the form ID `UXuY0A`. Uses stored Typeform API credentials.  
    - **Expressions/Variables:** Outputs the form response fields matching the Google Sheet columns.  
    - **Inputs:** None (trigger node).  
    - **Outputs:** Emits new submission data as JSON.  
    - **Version:** n8n Typeform Trigger node v1.  
    - **Potential Failures:**  
      - Authentication failures if Typeform API credentials are invalid.  
      - Network or API rate limiting.  
      - Form ID changes or deletion in Typeform.  
    - **Sub-workflow:** None.

#### 1.2 Data Logging

- **Overview:**  
  Appends the received form data to a Google Sheet named "Problems" in the specified spreadsheet, ensuring all reports are logged for tracking.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  

  - **Google Sheets**  
    - **Type and Role:** Append operation node; writes form data into Google Sheets.  
    - **Configuration:**  
      - Spreadsheet ID: `17fzSFl1BZ1njldTfp5lvh8HtS0-pNXH66b7qGZIiGRU`  
      - Sheet and range: `Problems!A:D`  
      - Operation: Append  
      - Uses Google API credentials configured in n8n.  
    - **Expressions/Variables:** Takes input from Typeform Trigger node. Data fields correspond exactly to columns: Name, Email, Severity, Problem.  
    - **Input:** Receives new submission data from Typeform Trigger.  
    - **Output:** Passes appended row data downstream for evaluation.  
    - **Version:** Google Sheets node v1.  
    - **Potential Failures:**  
      - Authentication errors due to invalid Google API credentials.  
      - Permission errors if the sheet or spreadsheet is inaccessible or protected.  
      - Data formatting issues if input data does not match expected columns.  
      - Network/API timeouts.  
    - **Sub-workflow:** None.

#### 1.3 Severity Evaluation

- **Overview:**  
  Evaluates the severity score from the appended Google Sheet data. Determines the flow: if severity is greater than 7, it is treated as very severe; otherwise, it is less severe.

- **Nodes Involved:**  
  - IF

- **Node Details:**  

  - **IF**  
    - **Type and Role:** Conditional node; branches the workflow based on severity value.  
    - **Configuration:**  
      - Condition: Number comparison  
      - Checks if `Severity` value from Google Sheets output data is greater than 7.  
    - **Expressions/Variables:** Uses expression: `={{$node["Google Sheets"].data["Severity"]}}`  
    - **Input:** Receives data from Google Sheets node.  
    - **Outputs:**  
      - Output 0 (true branch): Severity > 7 → triggers Slack notification.  
      - Output 1 (false branch): Severity ≤ 7 → triggers email notification.  
    - **Version:** n8n IF node v1.  
    - **Potential Failures:**  
      - Expression evaluation failure if severity data is missing or malformatted.  
      - Numeric comparison errors if severity is not a number.  
    - **Sub-workflow:** None.

#### 1.4 Conditional Notifications

- **Overview:**  
  Sends notifications based on the severity condition. Very severe problems trigger a Slack message in the `problems` channel, while less severe problems generate an email notification.

- **Nodes Involved:**  
  - Slack  
  - Send Email

- **Node Details:**  

  - **Slack**  
    - **Type and Role:** Message posting node; sends alert messages to Slack channel.  
    - **Configuration:**  
      - Text includes Email, Name, Severity, and Problem details pulled from IF node data.  
      - Channel: `problems`  
      - Uses Slack API credentials stored in n8n.  
    - **Expressions/Variables:**  
      - Text uses expressions like `=Email: {{$node["IF"].data["Email"]}}` etc.  
    - **Input:** True branch output from IF node.  
    - **Output:** None (end node).  
    - **Version:** Slack node v1.  
    - **Potential Failures:**  
      - Authentication errors if Slack credentials expire or are invalid.  
      - Channel access errors if channel does not exist or bot lacks permission.  
      - Rate limits or network failures.  
    - **Sub-workflow:** None.

  - **Send Email**  
    - **Type and Role:** Email sending node; sends email notifications for less severe problems.  
    - **Configuration:**  
      - Dynamic email body including Email, Name, Severity, and Problem data from IF node.  
      - Subject: "User Reported Problem"  
      - To and From email addresses must be configured.  
      - Uses SMTP credentials configured in n8n.  
    - **Expressions/Variables:** Same dynamic expressions as Slack node for body text.  
    - **Input:** False branch output from IF node.  
    - **Output:** None (end node).  
    - **Version:** Email Send node v1.  
    - **Potential Failures:**  
      - SMTP authentication or connection failures.  
      - Invalid or missing To/From email addresses.  
      - Email formatting errors.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name         | Node Type              | Functional Role                  | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                       |
|-------------------|------------------------|--------------------------------|---------------------|--------------------------|------------------------------------------------------------------------------------------------------------------|
| Typeform Trigger  | Typeform Trigger       | Receives new Typeform submissions | None                | Google Sheets             |                                                                                                                  |
| Google Sheets     | Google Sheets          | Appends form data to Google Sheet | Typeform Trigger    | IF                       |                                                                                                                  |
| IF                | IF Condition           | Checks severity to choose notification path | Google Sheets    | Slack (true branch), Send Email (false branch) |                                                                                                                  |
| Slack             | Slack                  | Sends Slack message for severe issues | IF (true branch)    | None                     |                                                                                                                  |
| Send Email        | Email Send             | Sends email notification for less severe issues | IF (false branch)   | None                     |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger node**  
   - Node type: Typeform Trigger  
   - Set form ID to `UXuY0A` (replace with your actual form ID).  
   - Configure Typeform API credentials.  
   - Position the node as the workflow entry point.

2. **Add Google Sheets node**  
   - Node type: Google Sheets  
   - Operation: Append  
   - Spreadsheet ID: `17fzSFl1BZ1njldTfp5lvh8HtS0-pNXH66b7qGZIiGRU`  
   - Range: `Problems!A:D`  
   - Configure Google API credentials.  
   - Connect Typeform Trigger node’s output to this node’s input.

3. **Add IF node**  
   - Node type: IF  
   - Condition: Number > 7  
   - Value 1: Expression → `{{$node["Google Sheets"].data["Severity"]}}`  
   - Value 2: `7`  
   - Connect Google Sheets node’s output to IF node’s input.

4. **Add Slack node**  
   - Node type: Slack  
   - Channel: `problems`  
   - Text:  
     ```
     Email: {{$node["IF"].data["Email"]}}
     Name: {{$node["IF"].data["Name"]}}
     Severity: {{$node["IF"].data["Severity"]}}

     Problem:
     {{$node["IF"].data["Problem"]}}
     ```  
   - Configure Slack API credentials.  
   - Connect IF node’s true output (severity > 7) to Slack node’s input.

5. **Add Send Email node**  
   - Node type: Email Send  
   - Subject: `User Reported Problem`  
   - To Email: (set appropriate recipient email)  
   - From Email: (set valid sender email)  
   - Text body same as Slack text (with expressions referencing IF node).  
   - Configure SMTP credentials.  
   - Connect IF node’s false output (severity ≤ 7) to Send Email node’s input.

6. **Verify all connections:**  
   - Typeform Trigger → Google Sheets → IF → Slack (true branch)  
   - IF → Send Email (false branch)

7. **Test workflow:**  
   - Submit a Typeform entry with different severity values to verify Slack and email branches trigger correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The Google Sheet must have columns named exactly: Name, Email, Severity, Problem in the "Problems" sheet.    | [Example Sheet](https://docs.google.com/spreadsheets/d/17fzSFl1BZ1njldTfp5lvh8HtS0-pNXH66b7qGZIiGRU) |
| Typeform question names should exactly match the Google Sheet column names for correct data mapping.          |                                                                                                     |
| Slack channel `problems` must exist and the API credentials must have permission to post messages there.      |                                                                                                     |
| Ensure SMTP credentials are correctly configured to allow email sending from the workflow environment.        |                                                                                                     |

---

This structured document provides all necessary details for understanding, reproducing, and maintaining the "Send Typeform results to Google Sheet, Slack and email" workflow in n8n.