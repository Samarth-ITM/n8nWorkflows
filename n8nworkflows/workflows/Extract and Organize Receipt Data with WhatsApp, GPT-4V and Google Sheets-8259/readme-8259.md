Extract and Organize Receipt Data with WhatsApp, GPT-4V and Google Sheets

https://n8nworkflows.xyz/workflows/extract-and-organize-receipt-data-with-whatsapp--gpt-4v-and-google-sheets-8259


# Extract and Organize Receipt Data with WhatsApp, GPT-4V and Google Sheets

### 1. Workflow Overview

This workflow automates client nurture email campaigns and testimonial collection using data from Notion, scheduled or manual triggers, and notification channels like email and Telegram. It targets businesses managing client relationships and seeking to automate milestone-based communications and testimonial gathering.

The workflow is logically divided into two main functional blocks:

- **1.1 Client Nurture Email Automation:**  
  This block fetches client data from Notion, calculates milestone days since client onboarding, evaluates which milestone emails to send (Day 7, 30, or 60), and dispatches corresponding emails followed by Telegram notifications.

- **1.2 Testimonial Collection and Notification:**  
  This block captures incoming testimonials via a webhook, parses the testimonial data, sends Telegram notifications about the new testimonial, and responds to the webhook to acknowledge receipt.

---

### 2. Block-by-Block Analysis

#### 2.1 Client Nurture Email Automation

- **Overview:**  
  Handles scheduled or manual triggering of client follow-up emails based on client milestones stored in Notion. It sends milestone-specific emails at Day 7, Day 30, and Day 60 after client onboarding and notifies via Telegram after each email dispatch.

- **Nodes Involved:**  
  Manual Trigger, Schedule Trigger, Get Today Date (Code), Get Notion Clients, Calculate Milestone Days (Code), IF Day 7, IF Day 30, IF Day 60, Send Day 7 Email, Send Day 30 Email, Send Day 60 Email, Notify via Telegram

- **Node Details:**

  - **Manual Trigger**  
    - *Type:* Trigger  
    - *Role:* Allows manual initiation of the workflow (e.g., for testing or on-demand execution).  
    - *Configuration:* Default; no parameters.  
    - *Input:* None  
    - *Output:* Starts flow to "Get Today Date" node.  
    - *Edge Cases:* None significant; manual trigger depends on user action.

  - **Schedule Trigger**  
    - *Type:* Trigger  
    - *Role:* Automates periodic execution (e.g., daily) of the workflow to send emails without manual intervention.  
    - *Configuration:* Default schedule settings (not explicitly detailed in JSON).  
    - *Input:* None  
    - *Output:* Starts flow to "Get Today Date" node.  
    - *Edge Cases:* Scheduling misconfiguration could cause missed runs.

  - **Get Today Date**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Generates current date/time to be used for milestone calculations.  
    - *Configuration:* Likely returns current date in a format compatible with downstream nodes.  
    - *Input:* From triggers  
    - *Output:* Passes current date to "Get Notion Clients".  
    - *Edge Cases:* Timezone differences could cause date calculation discrepancies.

  - **Get Notion Clients**  
    - *Type:* Notion node  
    - *Role:* Retrieves client data from Notion database, presumably including onboarding dates.  
    - *Configuration:* Requires Notion credentials and database ID (not visible in JSON).  
    - *Input:* Current date info  
    - *Output:* Client list to "Calculate Milestone Days".  
    - *Edge Cases:* Notion API rate limits or auth errors; incomplete client data can cause failures.

  - **Calculate Milestone Days**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Calculates the number of days elapsed since each client's onboarding date to determine which milestone emails to send.  
    - *Configuration:* Custom code comparing today's date with client onboarding dates.  
    - *Input:* Client data from Notion  
    - *Output:* Routes clients to IF nodes for milestone evaluation.  
    - *Edge Cases:* Missing or malformed dates can cause errors; date format inconsistencies.

  - **IF Day 7, IF Day 30, IF Day 60**  
    - *Type:* If nodes  
    - *Role:* Check if the client matches the milestone day (exactly 7, 30, or 60 days since onboarding).  
    - *Configuration:* Conditional expressions comparing calculated days to milestone values.  
    - *Input:* Client data with milestone day info.  
    - *Output:* True path to respective "Send Day X Email" nodes; false path ends or discards.  
    - *Edge Cases:* Timezone or calculation errors may route clients incorrectly.

  - **Send Day 7 Email, Send Day 30 Email, Send Day 60 Email**  
    - *Type:* Email Send nodes  
    - *Role:* Dispatch milestone-specific nurture emails to clients.  
    - *Configuration:* Uses configured SMTP or email service credentials; email content specific to milestone day.  
    - *Input:* Client data passing IF condition.  
    - *Output:* Passes success to "Notify via Telegram".  
    - *Edge Cases:* Email service downtime, invalid email addresses, or quota limits.

  - **Notify via Telegram**  
    - *Type:* Telegram node  
    - *Role:* Sends a Telegram message to notify about the sent milestone email, useful for audit or alerting.  
    - *Configuration:* Requires Telegram bot token and chat ID.  
    - *Input:* Triggered after each email sent.  
    - *Output:* None (endpoint).  
    - *Edge Cases:* Telegram API limits or invalid credentials.

---

#### 2.2 Testimonial Collection and Notification

- **Overview:**  
  Listens for incoming testimonial submissions via webhook, parses the testimonial content, sends a Telegram notification about the new testimonial, and responds to the webhook sender confirming receipt.

- **Nodes Involved:**  
  Testimonial Webhook, Parse Testimonial (Code), Notify Testimonial (Telegram), Respond to Webhook

- **Node Details:**

  - **Testimonial Webhook**  
    - *Type:* Webhook  
    - *Role:* Receives incoming HTTP requests containing testimonial data from external sources (e.g., a form or bot).  
    - *Configuration:* HTTP method and path configured to accept testimonial submissions (not detailed in JSON).  
    - *Input:* External HTTP request.  
    - *Output:* Passes raw testimonial data to "Parse Testimonial".  
    - *Edge Cases:* Webhook URL exposure risk, malformed requests, request authorization missing.

  - **Parse Testimonial**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Extracts and formats testimonial information from raw webhook payload.  
    - *Configuration:* Custom code parsing JSON or form data to structured format.  
    - *Input:* Raw webhook data.  
    - *Output:* Cleaned testimonial data to "Notify Testimonial".  
    - *Edge Cases:* Unexpected payload structure causing code errors.

  - **Notify Testimonial**  
    - *Type:* Telegram node  
    - *Role:* Sends Telegram notification alerting about the new testimonial received.  
    - *Configuration:* Uses Telegram bot credentials and target chat ID.  
    - *Input:* Parsed testimonial data.  
    - *Output:* Passes to "Respond to Webhook".  
    - *Edge Cases:* Telegram API problems, message formatting errors.

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends HTTP response back to original webhook caller confirming receipt and processing.  
    - *Configuration:* Usually sends a 200 OK with a message.  
    - *Input:* Triggered after notification.  
    - *Output:* Ends workflow.  
    - *Edge Cases:* Response delay or failure could cause retries or webhook timeouts.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                  | Input Node(s)          | Output Node(s)          | Sticky Note |
|---------------------|---------------------|-------------------------------------------------|------------------------|-------------------------|-------------|
| Sticky Note         | Sticky Note         | Informational / Decorative                       | -                      | -                       |             |
| Sticky Note1        | Sticky Note         | Informational / Decorative                       | -                      | -                       |             |
| Manual Trigger      | Trigger             | Manual start of client nurture emails            | -                      | Get Today Date           |             |
| Schedule Trigger    | Trigger             | Scheduled start of client nurture emails         | -                      | Get Today Date           |             |
| Get Today Date      | Code                | Generate current date for milestone calculation  | Manual Trigger, Schedule Trigger | Get Notion Clients     |             |
| Get Notion Clients  | Notion              | Fetch clients data from Notion                    | Get Today Date          | Calculate Milestone Days |             |
| Calculate Milestone Days | Code            | Calculate days since client onboarding            | Get Notion Clients      | IF Day 7, IF Day 30, IF Day 60 |       |
| IF Day 7            | If                  | Check if day 7 milestone reached                  | Calculate Milestone Days| Send Day 7 Email         |             |
| IF Day 30           | If                  | Check if day 30 milestone reached                 | Calculate Milestone Days| Send Day 30 Email        |             |
| IF Day 60           | If                  | Check if day 60 milestone reached                 | Calculate Milestone Days| Send Day 60 Email        |             |
| Send Day 7 Email    | Email Send          | Send nurture email for day 7 milestone            | IF Day 7                | Notify via Telegram      |             |
| Send Day 30 Email   | Email Send          | Send nurture email for day 30 milestone           | IF Day 30               | Notify via Telegram      |             |
| Send Day 60 Email   | Email Send          | Send nurture email for day 60 milestone           | IF Day 60               | Notify via Telegram      |             |
| Notify via Telegram | Telegram            | Notify milestone email sent via Telegram          | Send Day 7, 30, 60 Email| -                       |             |
| Testimonial Webhook | Webhook             | Receive testimonial submissions                    | -                      | Parse Testimonial        |             |
| Parse Testimonial   | Code                | Parse testimonial data from webhook payload       | Testimonial Webhook     | Notify Testimonial       |             |
| Notify Testimonial  | Telegram            | Notify new testimonial received via Telegram      | Parse Testimonial       | Respond to Webhook       |             |
| Respond to Webhook  | Respond to Webhook  | Respond to testimonial webhook sender              | Notify Testimonial      | -                       |             |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create Trigger Nodes**  
- Add a **Manual Trigger** node with default settings for manual workflow start.  
- Add a **Schedule Trigger** node configured to run periodically (e.g., daily at a set time).

**Step 2: Add Date Calculation Node**  
- Add a **Code** node named "Get Today Date".  
- Configure it to output the current date/time in ISO format or desired format for date calculations.

**Step 3: Connect Triggers to Date Node**  
- Connect both "Manual Trigger" and "Schedule Trigger" nodes to "Get Today Date".

**Step 4: Fetch Clients from Notion**  
- Add a **Notion** node named "Get Notion Clients".  
- Configure it with valid Notion credentials and select the database containing client data.  
- Connect "Get Today Date" output to this node.

**Step 5: Calculate Milestone Days**  
- Add a **Code** node named "Calculate Milestone Days".  
- Implement JavaScript to calculate days elapsed since each client's onboarding date. For each client, output a field like `daysSinceOnboarding`.  
- Connect "Get Notion Clients" output here.

**Step 6: Add Conditional IF Nodes for Milestones**  
- Add three **If** nodes named "IF Day 7", "IF Day 30", and "IF Day 60".  
- Configure each to check if `daysSinceOnboarding` equals 7, 30, or 60 respectively.  
- Connect "Calculate Milestone Days" output to all three IF nodes.

**Step 7: Configure Email Send Nodes**  
- Add three **Email Send** nodes named "Send Day 7 Email", "Send Day 30 Email", and "Send Day 60 Email".  
- Configure these with SMTP or email service credentials.  
- Prepare email templates/content specific to each milestone.  
- Connect the true output from each IF node to the corresponding Email Send node.

**Step 8: Add Telegram Notification Node**  
- Add a **Telegram** node named "Notify via Telegram".  
- Configure with Telegram Bot API credentials and target chat ID.  
- Connect outputs of all three Email Send nodes to this Telegram node.

**Step 9: Build Testimonial Collection Sub-Workflow**

- Add a **Webhook** node named "Testimonial Webhook".  
- Configure HTTP method and path to accept incoming testimonials.  
- Add a **Code** node "Parse Testimonial" to extract testimonial details from the webhook payload.  
- Connect "Testimonial Webhook" output to "Parse Testimonial".

- Add a **Telegram** node "Notify Testimonial".  
- Configure with Telegram bot credentials and chat ID.  
- Connect "Parse Testimonial" output here.

- Add a **Respond to Webhook** node.  
- Configure a simple acknowledgment response (e.g., HTTP 200 with a JSON message).  
- Connect "Notify Testimonial" output here.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                             |
|------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| The workflow demonstrates integration between Notion, email systems, Telegram, and webhook services. | Useful for automating client communication and feedback collection.                         |
| Telegram notifications provide real-time alerts for milestone emails sent and testimonials received. | Ensures transparency and monitoring for workflow operators.                               |
| For Notion integration, ensure API access is properly configured and the correct database IDs are used.| https://developers.notion.com/                                                             |
| Email nodes require valid SMTP or email service credentials (e.g., Gmail, Outlook).                   | Refer to n8n documentation on Email node credential setup: https://docs.n8n.io/nodes/n8n-nodes-base.emailSend/ |
| Webhook node should be secured appropriately if exposed publicly (e.g., with authentication or IP whitelisting). | Prevents unauthorized testimonial submissions.                                            |

---

**Disclaimer:** All content and data handled by this workflow are legal and publicly accessible. No illegal, offensive, or protected content is processed. This documentation is derived solely from the provided n8n workflow JSON.