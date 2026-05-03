Schedule Weekly Facebook Posts from Google Sheets Every Sunday

https://n8nworkflows.xyz/workflows/schedule-weekly-facebook-posts-from-google-sheets-every-sunday-7977


# Schedule Weekly Facebook Posts from Google Sheets Every Sunday

### 1. Workflow Overview

This workflow automates client nurture email campaigns based on milestone days (7, 30, 60) tracked in a Notion database, sending personalized emails accordingly and notifying via Telegram. It also includes a webhook to receive client testimonials, parse them, notify via Telegram, and respond to the webhook call. The workflow is designed to run daily at 9 AM automatically or can be triggered manually.

**Logical Blocks:**

- **1.1 Triggers & Configuration:** Manual and scheduled trigger nodes, and a configuration node.
- **1.2 Milestone Email Processing:** Fetches client data from Notion, calculates milestone days, filters for milestone days (7, 30, 60), sends appropriate emails, and notifies via Telegram.
- **1.3 Testimonial Handling:** Receives testimonial data via webhook, parses it, sends Telegram notification, and responds to the webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers & Configuration

**Overview:**  
Handles workflow initiation either manually or on a daily schedule at 9 AM. Also includes a placeholder configuration node for workflow variables or credentials.

**Nodes Involved:**  
- Manual Trigger  
- Daily at 9 AM (Schedule Trigger)  
- Configuration (Set)  

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger  
  - *Role:* Allows manual workflow start for testing or immediate execution.  
  - *Config:* No parameters configured; fires immediately on manual activation.  
  - *Input:* None  
  - *Output:* To "Get Today's Date" node.  
  - *Edge Cases:* None specific; user-dependent.

- **Daily at 9 AM**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers workflow every day at 9:00 AM to perform milestone checks.  
  - *Config:* Default daily schedule at 9:00 AM.  
  - *Input:* None  
  - *Output:* To "Get Today's Date" node.  
  - *Edge Cases:* Timezone misconfiguration can cause timing issues.

- **Configuration**  
  - *Type:* Set  
  - *Role:* Placeholder for storing static configuration or variables used throughout the workflow.  
  - *Config:* Empty, intended for future or external configuration.  
  - *Input:* None  
  - *Output:* None connected in current workflow.  
  - *Edge Cases:* None.

---

#### 1.2 Milestone Email Processing

**Overview:**  
Fetches client data from the Notion database, calculates how many days have passed since client milestone dates, filters clients at specific milestone days (7, 30, 60), sends personalized emails accordingly, and notifies a Telegram chat for each email sent.

**Nodes Involved:**  
- Get Today's Date (Code)  
- Query Notion Database (HTTP Request)  
- Calculate Milestone Days (Code)  
- Is Day 7? (If)  
- Is Day 30? (If)  
- Is Day 60? (If)  
- Send Day 7 Email (Email Send)  
- Send Day 30 Email (Email Send)  
- Send Day 60 Email (Email Send)  
- Notify via Telegram (Telegram)  

**Node Details:**

- **Get Today's Date**  
  - *Type:* Code  
  - *Role:* Generates the current date in the desired format for use in the workflow.  
  - *Config:* Uses JavaScript to create today's date, possibly in ISO or formatted string.  
  - *Input:* From triggers.  
  - *Output:* To "Query Notion Database".  
  - *Edge Cases:* Timezone mismatch can cause off-by-one-day errors.

- **Query Notion Database**  
  - *Type:* HTTP Request  
  - *Role:* Queries the Notion API to retrieve client records and milestone dates.  
  - *Config:* Configured with Notion API credentials and parameters to filter or fetch relevant database entries.  
  - *Input:* From "Get Today's Date".  
  - *Output:* To "Calculate Milestone Days".  
  - *Edge Cases:* API authentication failures, rate limits, or malformed query parameters.

- **Calculate Milestone Days**  
  - *Type:* Code  
  - *Role:* Calculates the number of days elapsed since each client's milestone date, adding this information to the dataset.  
  - *Config:* JavaScript code processing Notion data JSON, computing day differences.  
  - *Input:* From "Query Notion Database".  
  - *Output:* To three If nodes: "Is Day 7?", "Is Day 30?", "Is Day 60?".  
  - *Edge Cases:* Null or missing date fields, incorrect date formats.

- **Is Day 7? / Is Day 30? / Is Day 60?**  
  - *Type:* If  
  - *Role:* Filters clients whose milestone day matches 7, 30, or 60 respectively.  
  - *Config:* Expression checking if calculated milestone days equal 7, 30, or 60.  
  - *Input:* From "Calculate Milestone Days".  
  - *Output:* To corresponding "Send Day X Email" nodes if true.  
  - *Edge Cases:* Type coercion issues, clients with multiple records.

- **Send Day 7 Email / Send Day 30 Email / Send Day 60 Email**  
  - *Type:* Email Send  
  - *Role:* Sends personalized milestone emails to clients on milestone days.  
  - *Config:* SMTP credentials configured; email templates likely use variables from client data.  
  - *Input:* From respective If nodes.  
  - *Output:* To "Notify via Telegram".  
  - *Edge Cases:* SMTP authentication failure, invalid email addresses, template errors.

- **Notify via Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends notification messages to a Telegram chat after each milestone email is sent.  
  - *Config:* Telegram Bot API credentials configured; message content likely includes milestone info.  
  - *Input:* From all milestone email nodes.  
  - *Output:* None.  
  - *Edge Cases:* Telegram API errors, network issues.

---

#### 1.3 Testimonial Handling

**Overview:**  
Handles incoming client testimonials through a webhook, parses testimonial data, sends a Telegram notification about the new testimonial, and responds to the webhook request.

**Nodes Involved:**  
- Testimonial Webhook (Webhook)  
- Parse Testimonial Data (Code)  
- Notify New Testimonial (Telegram)  
- Respond to Webhook (Respond to Webhook)  

**Node Details:**

- **Testimonial Webhook**  
  - *Type:* Webhook  
  - *Role:* Receives HTTP POST requests containing testimonial data from external sources.  
  - *Config:* Webhook ID set to "testimonial-webhook"; exposed endpoint URL.  
  - *Input:* External HTTP request.  
  - *Output:* To "Parse Testimonial Data".  
  - *Edge Cases:* Invalid data payload, unauthorized calls (no auth configured).

- **Parse Testimonial Data**  
  - *Type:* Code  
  - *Role:* Extracts and formats relevant fields from the testimonial payload for downstream usage.  
  - *Config:* JavaScript parsing JSON body, validation, and preparing message content.  
  - *Input:* From webhook.  
  - *Output:* To "Notify New Testimonial".  
  - *Edge Cases:* Malformed or incomplete testimonial data.

- **Notify New Testimonial**  
  - *Type:* Telegram  
  - *Role:* Sends a Telegram notification alerting about the new testimonial received.  
  - *Config:* Uses Telegram bot credentials; message includes parsed testimonial details.  
  - *Input:* From "Parse Testimonial Data".  
  - *Output:* To "Respond to Webhook".  
  - *Edge Cases:* Telegram API failure or throttling.

- **Respond to Webhook**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP 200 OK response back to the webhook caller confirming receipt.  
  - *Config:* Default response settings, no body specified.  
  - *Input:* From "Notify New Testimonial".  
  - *Output:* None.  
  - *Edge Cases:* Delays or failures can cause webhook sender retries.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                          | Input Node(s)             | Output Node(s)                  | Sticky Note                          |
|-------------------------|--------------------|----------------------------------------|---------------------------|-------------------------------|------------------------------------|
| 📋 Workflow Documentation | Sticky Note        | Documentation placeholder               | None                      | None                          |                                    |
| ⚙️ Configuration Guide     | Sticky Note        | Configuration instructions placeholder | None                      | None                          |                                    |
| 🔧 Configuration           | Set                | Workflow configuration variables        | None                      | None                          |                                    |
| Manual Trigger            | Manual Trigger     | Manual initiation of workflow           | None                      | Get Today's Date              |                                    |
| Daily at 9 AM            | Schedule Trigger   | Scheduled daily workflow initiation     | None                      | Get Today's Date              |                                    |
| Get Today's Date         | Code               | Provides current date                    | Manual Trigger, Daily at 9 AM | Query Notion Database        |                                    |
| Query Notion Database    | HTTP Request       | Fetches client milestone data from Notion | Get Today's Date           | Calculate Milestone Days      |                                    |
| Calculate Milestone Days | Code               | Calculates elapsed milestone days       | Query Notion Database       | Is Day 7?, Is Day 30?, Is Day 60? |                                    |
| Is Day 7?                | If                 | Filters for clients on day 7 milestone  | Calculate Milestone Days    | Send Day 7 Email              |                                    |
| Is Day 30?               | If                 | Filters for clients on day 30 milestone | Calculate Milestone Days    | Send Day 30 Email             |                                    |
| Is Day 60?               | If                 | Filters for clients on day 60 milestone | Calculate Milestone Days    | Send Day 60 Email             |                                    |
| Send Day 7 Email         | Email Send         | Sends milestone day 7 email              | Is Day 7?                  | Notify via Telegram           |                                    |
| Send Day 30 Email        | Email Send         | Sends milestone day 30 email             | Is Day 30?                 | Notify via Telegram           |                                    |
| Send Day 60 Email        | Email Send         | Sends milestone day 60 email             | Is Day 60?                 | Notify via Telegram           |                                    |
| Notify via Telegram      | Telegram           | Sends Telegram notification after emails | Send Day 7/30/60 Email     | None                         |                                    |
| Testimonial Webhook      | Webhook            | Receives testimonial submissions         | None                      | Parse Testimonial Data        |                                    |
| Parse Testimonial Data   | Code               | Parses and formats testimonial data      | Testimonial Webhook        | Notify New Testimonial        |                                    |
| Notify New Testimonial   | Telegram           | Sends Telegram notification for testimonial | Parse Testimonial Data     | Respond to Webhook            |                                    |
| Respond to Webhook       | Respond to Webhook | Sends HTTP response confirming receipt   | Notify New Testimonial      | None                         |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Type: Manual Trigger  
   - No parameters needed.  

2. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run daily at 09:00 AM (local timezone).  

3. **Create a Set node for configuration:**  
   - Type: Set  
   - Leave empty or add configuration variables if needed.  

4. **Create a Code node named "Get Today's Date":**  
   - Type: Code  
   - JavaScript to output the current date in ISO format or desired format. Example: `return [{ json: { today: new Date().toISOString().substring(0,10) } }];`  
   - Connect outputs of Manual Trigger and Schedule Trigger to this node.  

5. **Create an HTTP Request node named "Query Notion Database":**  
   - Type: HTTP Request  
   - Configure with Notion API credentials (OAuth or token).  
   - Set method to POST or GET according to Notion API docs to query the database.  
   - Use parameters/filters to fetch client records with milestone dates.  
   - Connect "Get Today's Date" output to this node.  

6. **Create a Code node named "Calculate Milestone Days":**  
   - Type: Code  
   - JavaScript code to iterate over Notion results, calculate days between milestone date and today, and add this field to each record.  
   - Connect "Query Notion Database" output to this node.  

7. **Create three If nodes named "Is Day 7?", "Is Day 30?", and "Is Day 60?":**  
   - Type: If  
   - For each, set condition to check if calculated milestone days equal 7, 30, or 60 respectively.  
   - Connect "Calculate Milestone Days" output to all three If nodes.  

8. **Create three Email Send nodes named "Send Day 7 Email", "Send Day 30 Email", and "Send Day 60 Email":**  
   - Type: Email Send  
   - Configure SMTP credentials (e.g., SMTP server, port, username, password).  
   - Set recipient email dynamically from client data.  
   - Create email templates personalized for each milestone day.  
   - Connect the `true` output of each If node to the corresponding Email Send node.  

9. **Create a Telegram node named "Notify via Telegram":**  
   - Type: Telegram  
   - Configure Telegram Bot API credentials (bot token).  
   - Set chat ID to send milestone notifications.  
   - Compose message templates summarizing sent emails.  
   - Connect outputs from all three Email Send nodes to this Telegram node.  

10. **Create a Webhook node named "Testimonial Webhook":**  
    - Type: Webhook  
    - Set webhook ID to "testimonial-webhook" or custom.  
    - Expose POST endpoint for external testimonial submission.  

11. **Create a Code node named "Parse Testimonial Data":**  
    - Type: Code  
    - Extract and validate testimonial fields from webhook payload.  
    - Format message for Telegram notification.  
    - Connect "Testimonial Webhook" output to this node.  

12. **Create a Telegram node named "Notify New Testimonial":**  
    - Type: Telegram  
    - Use same Telegram Bot credentials as above or separate bot.  
    - Send notification with parsed testimonial details.  
    - Connect "Parse Testimonial Data" output to this node.  

13. **Create a Respond to Webhook node named "Respond to Webhook":**  
    - Type: Respond to Webhook  
    - Default HTTP 200 OK response to confirm receipt.  
    - Connect "Notify New Testimonial" output to this node.  

14. **Connect the workflow nodes as described in the connections section:**  
    - Manual Trigger and Daily Trigger -> Get Today's Date -> Query Notion Database -> Calculate Milestone Days -> If nodes -> Email Sends -> Telegram notify.  
    - Testimonial Webhook -> Parse Testimonial Data -> Notify New Testimonial -> Respond to Webhook.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                      |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow uses Notion API for client milestone tracking and SMTP for email sending.          | Requires Notion API and SMTP credentials.           |
| Telegram notifications provide real-time monitoring of emails sent and testimonials received.    | Telegram Bot API credentials required.               |
| Webhook endpoint for testimonials allows seamless integration with external systems.             | Webhook ID: "testimonial-webhook"                    |
| For timezone consistency, ensure the server timezone matches the client data timezone.           | Important to avoid date calculation errors.          |
| The workflow is based on n8n Professional Template version 1.0.0 by n8n Professional Templates. | https://n8n.io/templates                              |

---

Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.