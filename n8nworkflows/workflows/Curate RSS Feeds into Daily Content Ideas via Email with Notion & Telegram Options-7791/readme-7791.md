Curate RSS Feeds into Daily Content Ideas via Email with Notion & Telegram Options

https://n8nworkflows.xyz/workflows/curate-rss-feeds-into-daily-content-ideas-via-email-with-notion---telegram-options-7791


# Curate RSS Feeds into Daily Content Ideas via Email with Notion & Telegram Options

### 1. Workflow Overview

This workflow, titled **"Graceful Content Sparks — RSS → Notion (n8n)"**, automates the curation of RSS feed content into daily content ideas, distributing them via email with optional integration into Notion and Telegram. It suits content creators, marketers, and social media managers who want to aggregate RSS content, refine it, and receive curated ideas daily without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Configuration Setup:** Initiates the workflow on a daily schedule or manually and sets user-specific parameters.
- **1.2 RSS Feed Processing:** Reads RSS feeds, builds URL items, collects and angles content, and trims content to prepare for dissemination.
- **1.3 Content Packaging & Distribution:** Builds an email digest and conditionally sends content to Notion and Telegram based on user preferences.
- **1.4 Error Handling:** Captures any errors during execution and notifies the workflow owner by email.

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & Configuration Setup

**Overview:**  
This block handles the initiation of the workflow either by a scheduled daily cron job or manual trigger and then sets up user-configured parameters necessary for subsequent processing.

**Nodes Involved:**  
- Daily Cron  
- Manual Trigger  
- Set: User Config  

**Node Details:**

- **Daily Cron**  
  - *Type:* Cron Trigger  
  - *Role:* Automatically triggers the workflow daily at a preset time.  
  - *Configuration:* Default daily schedule (specific time not detailed in JSON).  
  - *Connections:* Outputs to "Set: User Config".  
  - *Failure Modes:* Cron misconfiguration, timezone errors.  
  - *Version:* 1  

- **Manual Trigger**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow for testing or ad-hoc runs.  
  - *Configuration:* No parameters.  
  - *Connections:* Outputs to "Set: User Config".  
  - *Failure Modes:* None typical; manual action required.  
  - *Version:* 1  

- **Set: User Config**  
  - *Type:* Set Node  
  - *Role:* Defines user-specific configuration variables such as RSS feed URLs, email recipients, Notion and Telegram enable flags, and other parameters used downstream.  
  - *Configuration:* Variables set internally (not detailed in JSON) and used as expressions in later nodes.  
  - *Connections:* Outputs to "Function: Build URL Items".  
  - *Failure Modes:* Missing or malformed config values can cause downstream failures.  
  - *Version:* 2  

---

#### 2.2 RSS Feed Processing

**Overview:**  
This block constructs the list of RSS feed URLs, reads the feeds, processes and angles the collected content, and trims it to prepare a concise dataset for output.

**Nodes Involved:**  
- Function: Build URL Items  
- RSS: Read Feed  
- Function: Collect & Angle  
- Function: Trim  

**Node Details:**

- **Function: Build URL Items**  
  - *Type:* Function  
  - *Role:* Constructs an array of RSS feed URLs from the user config for the RSS feed reader node.  
  - *Configuration:* Uses expressions to dynamically build feed URLs list.  
  - *Connections:* Input from "Set: User Config", output to "RSS: Read Feed".  
  - *Failure Modes:* Empty or invalid URL arrays cause RSS node failure.  
  - *Version:* 2  

- **RSS: Read Feed**  
  - *Type:* RSS Feed Read  
  - *Role:* Fetches and parses RSS feed data from URLs provided.  
  - *Configuration:* Receives dynamic URLs from previous node, default polling settings.  
  - *Connections:* Input from "Function: Build URL Items", output to "Function: Collect & Angle".  
  - *Failure Modes:* Network issues, invalid RSS format, feed downtime.  
  - *Version:* 1  

- **Function: Collect & Angle**  
  - *Type:* Function  
  - *Role:* Processes raw RSS items to select relevant content pieces, possibly applying thematic angles or filters.  
  - *Configuration:* Custom JavaScript logic for filtering, sorting, or enhancing feed items.  
  - *Connections:* Input from "RSS: Read Feed", output to "Function: Trim".  
  - *Failure Modes:* Logic errors, empty input, data format inconsistencies.  
  - *Version:* 2  

- **Function: Trim**  
  - *Type:* Function  
  - *Role:* Trims or shortens content items to fit desired limits or formats for email and other outputs.  
  - *Configuration:* Defined trimming logic, e.g., character limits or summarization heuristics.  
  - *Connections:* Input from "Function: Collect & Angle", outputs to "Function: Build Email" and "If: Notion Enabled?".  
  - *Failure Modes:* Over-trimming resulting in loss of meaning, unexpected input structure.  
  - *Version:* 2  

---

#### 2.3 Content Packaging & Distribution

**Overview:**  
This block formats the curated content into an email digest and conditionally sends it via email, adds ideas to Notion, and optionally sends notifications via Telegram.

**Nodes Involved:**  
- Function: Build Email  
- Email: Send Digest  
- If: Notion Enabled?  
- Notion: Create Idea  
- If: Telegram Enabled?  
- Telegram: Ping  

**Node Details:**

- **Function: Build Email**  
  - *Type:* Function  
  - *Role:* Constructs the email content body with the curated and trimmed content items.  
  - *Configuration:* Custom JavaScript templates to format the digest.  
  - *Connections:* Input from "Function: Trim", output to "Email: Send Digest" and "If: Notion Enabled?".  
  - *Failure Modes:* Template errors, missing data.  
  - *Version:* 2  

- **Email: Send Digest**  
  - *Type:* Email Send  
  - *Role:* Sends the curated content digest via email to configured recipients.  
  - *Configuration:* Uses configured SMTP or email credentials; subject and recipients set via expressions.  
  - *Connections:* Input from "Function: Build Email".  
  - *Failure Modes:* Authentication failure, SMTP connection errors, invalid email addresses.  
  - *Version:* 2  

- **If: Notion Enabled?**  
  - *Type:* If Node  
  - *Role:* Checks user config flag to determine if Notion integration is enabled.  
  - *Configuration:* Boolean condition based on user config variables.  
  - *Connections:* Input from "Function: Trim", outputs to "Notion: Create Idea" (true) or nothing (false).  
  - *Failure Modes:* Misconfigured flag leads to skipping or unwanted execution.  
  - *Version:* 2  

- **Notion: Create Idea**  
  - *Type:* Notion Node  
  - *Role:* Creates new pages or entries in Notion to store curated content ideas.  
  - *Configuration:* Requires Notion API credentials, target database or page ID, and mapped content fields.  
  - *Connections:* Input from "If: Notion Enabled?".  
  - *Failure Modes:* Authentication errors, API rate limits, invalid Notion page/database IDs.  
  - *Version:* 2  

- **If: Telegram Enabled?**  
  - *Type:* If Node  
  - *Role:* Checks user config flag to determine if Telegram notification is enabled.  
  - *Configuration:* Boolean condition based on config.  
  - *Connections:* Input from "Email: Send Digest", outputs to "Telegram: Ping" (true) or nothing (false).  
  - *Failure Modes:* Misconfiguration can prevent notifications.  
  - *Version:* 2  

- **Telegram: Ping**  
  - *Type:* Telegram Node  
  - *Role:* Sends a notification message via Telegram bot to a configured chat or user.  
  - *Configuration:* Requires Telegram Bot API credentials, chat ID, and message content.  
  - *Connections:* Input from "If: Telegram Enabled?".  
  - *Failure Modes:* Bot token invalid, chat ID incorrect, network issues.  
  - *Version:* 1  

---

#### 2.4 Error Handling

**Overview:**  
This block captures any runtime errors generated anywhere in the workflow and sends an alert email to the workflow owner for prompt troubleshooting.

**Nodes Involved:**  
- On Error  
- On Error → Email Owner  

**Node Details:**

- **On Error**  
  - *Type:* Error Trigger  
  - *Role:* Globally listens for any errors in the workflow execution.  
  - *Configuration:* Default error trapping.  
  - *Connections:* Outputs to "On Error → Email Owner".  
  - *Failure Modes:* None (intended to catch errors).  
  - *Version:* 1  

- **On Error → Email Owner**  
  - *Type:* Email Send  
  - *Role:* Sends an error notification email to the designated owner email address with error details.  
  - *Configuration:* SMTP/email credentials configured; email subject and body contain error information.  
  - *Connections:* Input from "On Error".  
  - *Failure Modes:* SMTP failures, email address misconfiguration.  
  - *Version:* 2  

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)                 | Sticky Note                                                                                     |
|-----------------------|--------------------|------------------------------------|-----------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Sticky: README        | Sticky Note        | Documentation placeholder          |                       |                               |                                                                                                |
| Sticky: Prereqs + Schema | Sticky Note        | Documentation placeholder          |                       |                               |                                                                                                |
| Sticky: Setup Checklist | Sticky Note        | Documentation placeholder          |                       |                               |                                                                                                |
| Sticky: Testing & Behavior | Sticky Note        | Documentation placeholder          |                       |                               |                                                                                                |
| Sticky: Compliance + Troubleshooting | Sticky Note        | Documentation placeholder          |                       |                               |                                                                                                |
| Daily Cron            | Cron Trigger       | Daily scheduled workflow trigger   |                       | Set: User Config              |                                                                                                |
| Manual Trigger        | Manual Trigger     | Manual workflow trigger            |                       | Set: User Config              |                                                                                                |
| Set: User Config      | Set                | Define user parameters             | Daily Cron, Manual Trigger | Function: Build URL Items    |                                                                                                |
| Function: Build URL Items | Function           | Build array of feed URLs           | Set: User Config       | RSS: Read Feed                |                                                                                                |
| RSS: Read Feed        | RSS Feed Read      | Fetch and parse RSS feeds          | Function: Build URL Items | Function: Collect & Angle     |                                                                                                |
| Function: Collect & Angle | Function           | Filter and angle RSS content       | RSS: Read Feed         | Function: Trim                |                                                                                                |
| Function: Trim        | Function           | Trim content items for output      | Function: Collect & Angle | Function: Build Email, If: Notion Enabled? |                                                                                                |
| Function: Build Email | Function           | Create email digest content        | Function: Trim         | Email: Send Digest, If: Notion Enabled? |                                                                                                |
| If: Notion Enabled?   | If                 | Check if Notion integration active | Function: Trim, Function: Build Email | Notion: Create Idea (true)   |                                                                                                |
| Notion: Create Idea   | Notion             | Add content ideas to Notion        | If: Notion Enabled? (true) |                               |                                                                                                |
| Email: Send Digest    | Email Send         | Send curated content email         | Function: Build Email  | If: Telegram Enabled?         |                                                                                                |
| If: Telegram Enabled? | If                 | Check if Telegram integration active | Email: Send Digest    | Telegram: Ping (true)          |                                                                                                |
| Telegram: Ping        | Telegram           | Send Telegram notification         | If: Telegram Enabled?  |                               |                                                                                                |
| On Error              | Error Trigger      | Capture workflow execution errors  |                       | On Error → Email Owner        |                                                                                                |
| On Error → Email Owner | Email Send         | Notify owner on error              | On Error               |                               |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Cron** node named "Daily Cron" with a schedule to run daily at your preferred time.
   - Add a **Manual Trigger** node named "Manual Trigger" for manual invocation.

2. **Set User Configuration:**
   - Add a **Set** node named "Set: User Config".
   - Configure user settings such as:
     - RSS feed URLs (array or comma-separated string)
     - Email recipients and sender address
     - Flags for enabling Notion and Telegram integrations (boolean)
     - Notion database/page IDs and Telegram chat ID (if applicable)
   - Connect both "Daily Cron" and "Manual Trigger" nodes to this node.

3. **Build URL Items Array:**
   - Add a **Function** node named "Function: Build URL Items".
   - Write JavaScript code to parse the user-configured RSS feed URLs and output them as an array of URLs.
   - Connect "Set: User Config" to this node.

4. **Read RSS Feeds:**
   - Add an **RSS Feed Read** node named "RSS: Read Feed".
   - Configure it to accept dynamic URLs from the previous node's output.
   - Connect "Function: Build URL Items" to this node.

5. **Collect and Angle Content:**
   - Add a **Function** node named "Function: Collect & Angle".
   - Implement logic to filter, sort, or apply thematic angles to RSS feed items.
   - Connect "RSS: Read Feed" to this node.

6. **Trim Content:**
   - Add a **Function** node named "Function: Trim".
   - Write code to shorten or format content items to desired lengths.
   - Connect "Function: Collect & Angle" to this node.

7. **Build Email Digest:**
   - Add a **Function** node named "Function: Build Email".
   - Format the trimmed items into an email body template.
   - Connect "Function: Trim" to this node.

8. **Send Email Digest:**
   - Add an **Email Send** node named "Email: Send Digest".
   - Configure with SMTP or other email credentials.
   - Set recipient(s), subject, and body using expressions referencing "Function: Build Email" output.
   - Connect "Function: Build Email" to this node.

9. **Notion Integration:**
   - Add an **If** node named "If: Notion Enabled?".
   - Set condition to check the user config flag for Notion integration.
   - Connect "Function: Trim" to this node.
   - Add a **Notion** node named "Notion: Create Idea".
     - Configure with Notion API credentials and target database/page IDs.
     - Map fields to the trimmed content data.
   - Connect the "true" output of the If node to "Notion: Create Idea".

10. **Telegram Integration:**
    - Add an **If** node named "If: Telegram Enabled?".
    - Set condition to check the user config flag for Telegram integration.
    - Connect "Email: Send Digest" node to this node.
    - Add a **Telegram** node named "Telegram: Ping".
      - Configure with Telegram Bot API credentials and target chat ID.
      - Set message content referencing the curated content.
    - Connect the "true" output of the If node to "Telegram: Ping".

11. **Error Handling Setup:**
    - Add an **Error Trigger** node named "On Error".
    - Add an **Email Send** node named "On Error → Email Owner".
      - Configure to send error details to the owner's email.
    - Connect "On Error" node output to "On Error → Email Owner".

12. **Finalize Connections:**
    - Verify all node connections match the described flow.
    - Ensure credentials for Email, Notion, and Telegram nodes are properly attached.
    - Test manually using the "Manual Trigger" and observe outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Please attach Email, Notion, and Telegram credentials after importing the workflow; no secrets are included in the JSON export.                                   | Workflow import instructions                         |
| Ensure your RSS feed URLs are accessible and valid to avoid failures in feed reading.                                                                              | RSS Feed configuration best practice                 |
| Notion API rate limits may apply; handle with care when scaling workflow frequency or volume.                                                                     | https://developers.notion.com/reference/rate-limits |
| For troubleshooting errors, review the error email notifications sent to the configured owner address.                                                           | Error handling and notification                       |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.