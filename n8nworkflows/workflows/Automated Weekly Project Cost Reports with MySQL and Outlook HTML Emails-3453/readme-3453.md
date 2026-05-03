Automated Weekly Project Cost Reports with MySQL and Outlook HTML Emails

https://n8nworkflows.xyz/workflows/automated-weekly-project-cost-reports-with-mysql-and-outlook-html-emails-3453


# Automated Weekly Project Cost Reports with MySQL and Outlook HTML Emails

### 1. Workflow Overview

This workflow automates the generation and distribution of weekly project cost reports by querying a MySQL database for projects missing budgeted cost data and sending tailored HTML email notifications via Microsoft Outlook. It is designed primarily for finance or operations teams to stay updated on missing project cost information without manual effort.

The workflow logic is divided into these main blocks:

- **1.1 Schedule Trigger:** Initiates the workflow every Monday at 8 AM.
- **1.2 Data Retrieval (MySQL Query):** Queries the MySQL database to fetch counts of active external projects with zero budgeted cost, grouped by company and cost center.
- **1.3 Conditional Routing (Switch):** Routes the data based on the `default_cost_center` field to send customized emails for each cost center.
- **1.4 Email Notification (Microsoft Outlook Nodes):** Sends formatted HTML emails to designated recipients for each cost center group.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This node triggers the workflow automatically on a weekly schedule, specifically every Monday at 8 AM, ensuring timely report generation without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates workflow execution on a fixed weekly interval.  
  - **Configuration:**  
    - Interval set to trigger every 1 week.  
    - Trigger hour set to 8 AM.  
  - **Expressions/Variables:** None.  
  - **Input/Output:**  
    - No input; output connects to MySQL node.  
  - **Version Requirements:** n8n version supporting Schedule Trigger v1.2 or higher.  
  - **Potential Failures:**  
    - Misconfigured timezone could cause unexpected trigger times.  
    - Workflow disabled or paused would prevent execution.  

#### 1.2 Data Retrieval (MySQL Query)

- **Overview:**  
  Executes a SQL query against the MySQL database to retrieve the count of active, external projects with zero budgeted cost, grouped by company and cost center.

- **Nodes Involved:**  
  - MySQL

- **Node Details:**  
  - **Type:** MySQL Node  
  - **Role:** Connects to MySQL and runs a custom SQL query.  
  - **Configuration:**  
    - Connection via configured MySQL credentials (hostname, port, database, username, password).  
    - SQL Query:
      ```sql
      SELECT 
          company,
          cost_center AS default_cost_center,
          COUNT(*) AS project_count
      FROM 
          tabProject
      WHERE 
          status = 'Open' 
          AND project_type = 'External'
          AND is_active = 'Yes'
          AND budgeted_project_cost = 0
      GROUP BY 
          company, cost_center
      ORDER BY 
          company, project_count DESC;
      ```
  - **Expressions/Variables:** None inside the query; static SQL.  
  - **Input/Output:**  
    - Input from Schedule Trigger.  
    - Output JSON with fields: `company`, `default_cost_center`, `project_count`.  
  - **Version Requirements:** MySQL node version 2.4 or higher recommended.  
  - **Potential Failures:**  
    - Authentication errors if credentials are invalid.  
    - Query syntax errors if the database schema changes.  
    - Connection timeouts or network issues.  
    - Empty result sets if no projects match criteria.  

#### 1.3 Conditional Routing (Switch)

- **Overview:**  
  Routes the query results based on the `default_cost_center` value to different email sending nodes, enabling customized notifications per cost center.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  
  - **Type:** Switch Node  
  - **Role:** Branches workflow execution according to cost center values.  
  - **Configuration:**  
    - Uses strict string equality conditions on `{{ $json.default_cost_center }}`.  
    - Routes defined for:  
      - Output A: `"Cost Center A"`  
      - Output B: `"Cost Center B"`  
      - Output C: `"=COST CENTER C"` (note the leading `=` sign, possibly a misconfiguration)  
      - Output D: `"Cost Center D"`  
    - Output D is defined but has no connected downstream node.  
  - **Expressions/Variables:**  
    - Condition expressions use mustache syntax to evaluate `default_cost_center`.  
  - **Input/Output:**  
    - Input from MySQL node.  
    - Outputs connected to respective Microsoft Outlook nodes for A, B, C; D is unconnected.  
  - **Version Requirements:** Switch node version 3.2 or higher.  
  - **Potential Failures:**  
    - Case sensitivity may cause misrouting if cost center values differ in case.  
    - The condition for Output C includes an extra `=` sign which may cause no matches.  
    - Unhandled cost centers not matching any condition will be dropped silently.  
    - Missing output connection for Output D means emails for that cost center will not be sent.  

#### 1.4 Email Notification (Microsoft Outlook Nodes)

- **Overview:**  
  Sends HTML formatted emails via Microsoft Outlook to notify respective teams about missing budgeted cost projects per cost center.

- **Nodes Involved:**  
  - Microsoft Outlook6 (for Cost Center A)  
  - Microsoft Outlook1 (for Cost Center B)  
  - Microsoft Outlook7 (for Cost Center C)  

- **Node Details (each node):**  
  - **Type:** Microsoft Outlook Node  
  - **Role:** Sends email messages using Outlook with OAuth2 authentication.  
  - **Configuration:**  
    - Subject: `"Project Cost Missing"` or `"Projects Cost Missing"` (minor variation)  
    - Body: Custom HTML email with inline CSS styling, including:  
      - Header showing cost center name (`{{ $json.default_cost_center }}`)  
      - Body text mentioning the number of projects missing budgeted cost (`{{ $json.project_count }}`)  
      - Footer with sender signature "Amjid Ali" and "Automation Demo – n8n"  
    - Recipients: `"amjid@amjidali.com"` (placeholder, to be replaced with actual team emails)  
    - Body content type set to HTML.  
  - **Expressions/Variables:**  
    - Uses mustache tags to inject dynamic content from JSON fields.  
  - **Input/Output:**  
    - Input from Switch node outputs A, B, C respectively.  
    - No outputs (end of workflow branch).  
  - **Version Requirements:** Microsoft Outlook node version 2 or higher.  
  - **Potential Failures:**  
    - OAuth2 authentication errors if credentials expire or are misconfigured.  
    - Email sending failures due to network issues or Outlook service outages.  
    - Invalid recipient email addresses causing send errors.  
    - HTML rendering issues if email clients have strict policies.  

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                 | Input Node(s)     | Output Node(s)           | Sticky Note                                                                                  |
|---------------------|-------------------------|--------------------------------|-------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger        | Initiates workflow weekly      | -                 | MySQL                    | Runs every Monday at 8 AM to automate report generation                                      |
| MySQL               | MySQL                   | Queries project data           | Schedule Trigger  | Switch                   | Queries active external projects with zero budgeted cost                                    |
| Switch              | Switch                  | Routes by cost center          | MySQL             | Microsoft Outlook6, Microsoft Outlook1, Microsoft Outlook7 | Modify cost center values as needed; note possible misconfiguration in Output C condition    |
| Microsoft Outlook6  | Microsoft Outlook       | Sends email for Cost Center A  | Switch (A)        | -                        | Sends HTML email notification to Cost Center A team                                          |
| Microsoft Outlook1  | Microsoft Outlook       | Sends email for Cost Center B  | Switch (B)        | -                        | Sends HTML email notification to Cost Center B team                                          |
| Microsoft Outlook7  | Microsoft Outlook       | Sends email for Cost Center C  | Switch (C)        | -                        | Sends HTML email notification to Cost Center C team                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger every 1 week on Mondays at 8:00 AM.  
   - No credentials needed.  

2. **Create MySQL Node**  
   - Type: MySQL  
   - Configure credentials with MySQL connection details (hostname, port 3306 default, database name, username with SELECT permissions, password).  
   - Set operation to "Execute Query".  
   - Enter SQL query:
     ```sql
     SELECT 
         company,
         cost_center AS default_cost_center,
         COUNT(*) AS project_count
     FROM 
         tabProject
     WHERE 
         status = 'Open' 
         AND project_type = 'External'
         AND is_active = 'Yes'
         AND budgeted_project_cost = 0
     GROUP BY 
         company, cost_center
     ORDER BY 
         company, project_count DESC;
     ```
   - Connect Schedule Trigger output to MySQL node input.

3. **Create Switch Node**  
   - Type: Switch  
   - Add rules to check `default_cost_center` field with strict string equality:  
     - Output A: equals `"Cost Center A"`  
     - Output B: equals `"Cost Center B"`  
     - Output C: equals `"Cost Center C"` (correct the extra `=` sign from original)  
     - Output D: equals `"Cost Center D"` (optional, no downstream node needed unless you add one)  
   - Connect MySQL node output to Switch node input.

4. **Create Microsoft Outlook Nodes for Each Cost Center**  
   For each cost center (A, B, C):  
   - Create a Microsoft Outlook node.  
   - Configure OAuth2 credentials for Outlook account with send email permissions.  
   - Set email subject to `"Project Cost Missing"` or `"Projects Cost Missing"`.  
   - Set body content type to HTML.  
   - Use the following HTML template for the email body (replace placeholders where needed):
     ```html
     <!DOCTYPE html>
     <html lang="en">
     <head>
       <meta charset="UTF-8">
       <title>Missing Budgeted Cost Notification</title>
       <style>
         body { font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 0; }
         .email-container { max-width: 600px; margin: 20px auto; background-color: #ffffff; border-radius: 8px; overflow: hidden; }
         .email-header { background-color: #007BFF; color: #ffffff; padding: 20px; text-align: center; font-size: 18px; font-weight: bold; }
         .email-body { padding: 20px; font-size: 16px; color: #333333; }
         .email-body strong { color: #007BFF; }
         .email-footer { padding: 10px 20px; font-size: 14px; color: #555555; text-align: left; }
       </style>
     </head>
     <body>
       <div class="email-container">
         <div class="email-header">
           {{ $json.default_cost_center }} - Project Data Missing
         </div>
         <div class="email-body">
           Dear {{ $json.default_cost_center }} Team,<br><br>
           There are <strong>{{ $json.project_count }}</strong> active projects with missing <strong>Budgeted Cost</strong>.<br>
           Kindly coordinate with the <strong>Accounts Team</strong> to update the missing values for accurate tracking.<br><br>
           Your timely attention is appreciated.<br><br>
           Regards,
         </div>
         <div class="email-footer">
           <strong>Amjid Ali</strong><br>
           Automation Demo – n8n
         </div>
       </div>
     </body>
     </html>
     ```
   - Set recipient emails in `To` field (replace `"amjid@amjidali.com"` with actual team emails).  
   - Connect Switch outputs A, B, and C to the respective Microsoft Outlook nodes.

5. **Optional: Handle Unmatched Cost Centers**  
   - Consider adding a default branch or error handling for cost centers not matching any rule.

6. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by triggering manually or waiting for scheduled execution.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates practical automation without AI, saving hours weekly for finance teams. | Use case description and workflow purpose.                                                      |
| Modify cost center values in the Switch node to match your organization's naming conventions.    | Switch node configuration details.                                                              |
| Replace placeholder email addresses with actual recipients before deployment.                    | Outlook nodes recipient configuration.                                                          |
| HTML email formatting uses inline CSS for broad email client compatibility.                      | Email body content in Microsoft Outlook nodes.                                                  |
| Video walkthrough available at YouTube channel: https://youtube.com/@syncbricks                  | Demo & tutorial video link.                                                                      |
| Read more about workflow simplicity and value: https://syncbricks.com/why-simple-n8n-workflows-often-deliver-more-value-than-complex-ones/ | External blog link on workflow design philosophy.                                               |
| Created by Amjid Ali — see https://amjidali.com and https://lms.syncbricks.com/books/n8n/        | Creator credits and additional learning resources.                                              |
| OAuth2 credentials needed for Microsoft Outlook nodes to send emails.                            | Credential setup instructions for Outlook integration.                                          |
| MySQL user must have SELECT permissions on the target database and table.                        | Database credential requirements.                                                               |

---

This document fully describes the workflow structure, node configurations, and setup instructions to enable advanced users and AI agents to understand, reproduce, and maintain the workflow effectively.