Capture Gumroad sales, add buyer to MailerLite group, log to Google Sheets CRM

https://n8nworkflows.xyz/workflows/capture-gumroad-sales--add-buyer-to-mailerlite-group--log-to-google-sheets-crm-3676


# Capture Gumroad sales, add buyer to MailerLite group, log to Google Sheets CRM

### 1. Workflow Overview

This workflow automates the process of capturing new sales from a Gumroad store, adding the buyer as a subscriber to a MailerLite email group, and logging the sale details into a Google Sheets CRM. It is designed for online sellers who want to streamline customer onboarding and maintain an up-to-date CRM without manual data entry.

The workflow is logically divided into three main blocks:

- **1.1 Gumroad Sale Trigger:** Listens continuously for new completed sales on Gumroad.
- **1.2 MailerLite Subscriber Management:** Creates or updates the subscriber in MailerLite and assigns them to a specific subscriber group for targeted email sequences.
- **1.3 Google Sheets CRM Logging:** Appends the buyer and sale details as a new row in a Google Sheets spreadsheet acting as a CRM.

---

### 2. Block-by-Block Analysis

#### 2.1 Gumroad Sale Trigger

- **Overview:**  
  This block continuously monitors Gumroad for new sales events. When a sale is completed, it triggers the workflow to process the buyer’s information.

- **Nodes Involved:**  
  - Gumroad Sale Trigger  
  - Sticky Note (Trigger instructions)

- **Node Details:**

  - **Gumroad Sale Trigger**  
    - *Type:* Trigger node (gumroadTrigger)  
    - *Role:* Watches Gumroad account for new sales in real-time.  
    - *Configuration:* Uses Gumroad API credentials with an access token linked to a Gumroad application. Watches the "sale" resource.  
    - *Expressions/Variables:* Outputs sale data including buyer email, product name, sale timestamp, and country from the buyer’s IP.  
    - *Input/Output:* No input; outputs sale event data to the next node.  
    - *Version:* 1  
    - *Potential Failures:* Authentication errors if token invalid or revoked; network timeouts; API rate limits.  
    - *Sub-workflow:* None.

  - **Sticky Note (Trigger instructions)**  
    - Provides setup instructions and requirements for the Gumroad trigger.  
    - No technical role beyond documentation.

#### 2.2 MailerLite Subscriber Management

- **Overview:**  
  This block creates or updates the subscriber profile in MailerLite using the buyer’s email and country, then assigns the subscriber to a predefined MailerLite group to trigger email automation sequences.

- **Nodes Involved:**  
  - add subscriber to MailerLite  
  - Assign to group  
  - Sticky Note (MailerLite connection instructions)  
  - Sticky Note (Why assign to group)

- **Node Details:**

  - **add subscriber to MailerLite**  
    - *Type:* MailerLite node (mailerLite)  
    - *Role:* Adds or updates a subscriber in MailerLite with email and custom field for country.  
    - *Configuration:* Uses MailerLite API key credential. Email is dynamically set from Gumroad sale data. Custom field "country" is set from buyer’s IP country.  
    - *Expressions:* `email = {{$json.email}}`, `country = {{$json.ip_country}}`  
    - *Input/Output:* Receives sale data from Gumroad trigger; outputs subscriber data including subscriber ID.  
    - *Version:* 2  
    - *Potential Failures:* API key invalid; subscriber creation/update errors; missing required fields; network issues.

  - **Assign to group**  
    - *Type:* HTTP Request node  
    - *Role:* Assigns the subscriber to a specific MailerLite group by calling the MailerLite API endpoint.  
    - *Configuration:* POST request to `https://connect.mailerlite.com/api/subscribers/{{ $json.id }}/groups/152489030254069581` where `{{ $json.id }}` is subscriber ID from previous node. Uses MailerLite API credentials.  
    - *Expressions:* URL dynamically constructed with subscriber ID.  
    - *Input/Output:* Input from "add subscriber to MailerLite"; output goes to Google Sheets node.  
    - *Version:* 4.2  
    - *Potential Failures:* Invalid subscriber ID; group ID incorrect or deleted; API authentication errors; rate limiting.

  - **Sticky Note (MailerLite connection instructions)**  
    - Explains requirements for MailerLite account, API key, subscriber group creation, and how to obtain group ID.  
    - Provides links to MailerLite API docs for group assignment and listing groups.

  - **Sticky Note (Why assign to group)**  
    - Explains the rationale behind assigning subscribers to groups in MailerLite for triggering email sequences and automations.

#### 2.3 Google Sheets CRM Logging

- **Overview:**  
  This block appends the buyer’s sale data as a new row in a Google Sheets spreadsheet, serving as a CRM record.

- **Nodes Involved:**  
  - append row in CRM  
  - Sticky Note (Google Sheets instructions)

- **Node Details:**

  - **append row in CRM**  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row with sale data (date, email, country, product name) to a specified Google Sheets spreadsheet and worksheet.  
    - *Configuration:* Uses Google Sheets OAuth2 credentials. Spreadsheet and sheet selected by ID and gid respectively. Columns mapped explicitly from Gumroad sale data fields.  
    - *Expressions:*  
      - date: `={{ $('Gumroad Sale Trigger').item.json.sale_timestamp }}`  
      - email: `={{ $('Gumroad Sale Trigger').item.json.email }}`  
      - country: `={{ $('Gumroad Sale Trigger').item.json.ip_country }}`  
      - product name: `={{ $('Gumroad Sale Trigger').item.json.product_name }}`  
    - *Input/Output:* Input from "Assign to group" node; no output connections.  
    - *Version:* 4.5  
    - *Potential Failures:* Credential expiration; spreadsheet access errors; incorrect sheet or column mapping; API quota limits.

  - **Sticky Note (Google Sheets instructions)**  
    - Details requirements for Google Sheets API credentials and how to append rows with desired data.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                        | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                                         |
|---------------------------|---------------------|-------------------------------------|--------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Gumroad Sale Trigger       | gumroadTrigger      | Triggers workflow on new Gumroad sale | None                     | add subscriber to MailerLite | ## Trigger on a new Gumroad sale<br>Setup instructions and requirements for Gumroad trigger                                         |
| add subscriber to MailerLite | mailerLite          | Creates/updates subscriber in MailerLite | Gumroad Sale Trigger     | Assign to group          | ## Connection to [MailerLite](https://www.mailerlite.com/a/Kr9Yplim6ZhV) newsletter<br>Setup and API group assignment instructions  |
| Assign to group            | httpRequest         | Assigns subscriber to MailerLite group | add subscriber to MailerLite | append row in CRM        | See above MailerLite sticky note                                                                                                   |
| append row in CRM          | googleSheets        | Logs sale details into Google Sheets CRM | Assign to group          | None                     | ## Load into CRM<br>Google Sheets API setup and append row instructions                                                           |
| Sticky Note               | stickyNote          | Documentation for Gumroad trigger    | None                     | None                     | ## Trigger on a new Gumroad sale<br>Setup instructions and requirements for Gumroad trigger                                         |
| Sticky Note1              | stickyNote          | Documentation for MailerLite connection | None                     | None                     | ## Connection to MailerLite newsletter<br>Setup and API group assignment instructions                                               |
| Sticky Note2              | stickyNote          | Documentation for Google Sheets CRM  | None                     | None                     | ## Load into CRM<br>Google Sheets API setup and append row instructions                                                           |
| Sticky Note3              | stickyNote          | Explanation of MailerLite group usage | None                     | None                     | ## Why assign the subscriber to a group?<br>Explains benefits of MailerLite groups for automation                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gumroad Sale Trigger Node**  
   - Add a "Gumroad Trigger" node.  
   - Set resource to "sale".  
   - Authenticate with Gumroad API credentials (access token from Gumroad application).  
   - This node listens for new completed sales.

2. **Create MailerLite Subscriber Node**  
   - Add a "MailerLite" node.  
   - Set operation to create or update subscriber.  
   - Set email field to `{{$json.email}}` from Gumroad trigger output.  
   - Add custom field "country" with value `{{$json.ip_country}}`.  
   - Authenticate with MailerLite API key credential.

3. **Create HTTP Request Node to Assign Subscriber to Group**  
   - Add an "HTTP Request" node.  
   - Set method to POST.  
   - URL: `https://connect.mailerlite.com/api/subscribers/{{ $json.id }}/groups/152489030254069581`  
     - Replace `152489030254069581` with your MailerLite group ID.  
   - Authentication: Use the same MailerLite API key credential.  
   - Connect input from the MailerLite subscriber node.

4. **Create Google Sheets Node to Append Row**  
   - Add a "Google Sheets" node.  
   - Set operation to "append".  
   - Select your Google Sheets credentials (OAuth2).  
   - Choose the spreadsheet by ID and the worksheet by gid.  
   - Map columns explicitly:  
     - date: `={{ $('Gumroad Sale Trigger').item.json.sale_timestamp }}`  
     - email: `={{ $('Gumroad Sale Trigger').item.json.email }}`  
     - country: `={{ $('Gumroad Sale Trigger').item.json.ip_country }}`  
     - product name: `={{ $('Gumroad Sale Trigger').item.json.product_name }}`  
   - Connect input from the HTTP Request node.

5. **Connect Nodes in Sequence**  
   - Gumroad Sale Trigger → add subscriber to MailerLite → Assign to group → append row in CRM.

6. **Set Up Credentials**  
   - Gumroad API: Create an application in Gumroad, copy the access token, and add it to the Gumroad trigger node.  
   - MailerLite API: Generate API key from MailerLite Integrations and add it to both MailerLite nodes.  
   - Google Sheets OAuth2: Create OAuth2 credentials in Google Cloud Console, enable Sheets API, and add credentials to Google Sheets node.

7. **Create MailerLite Subscriber Group**  
   - In MailerLite dashboard, create a subscriber group (e.g., "Gumroad").  
   - Obtain the group ID by calling the MailerLite "list groups" API or from the dashboard.  
   - Replace the group ID in the HTTP Request node URL.

8. **Test the Workflow**  
   - Trigger a test sale in Gumroad.  
   - Verify subscriber creation and group assignment in MailerLite.  
   - Confirm new row appended in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Turn Gumroad buyers into loyal email subscribers and keep your CRM up-to-date.                                                   | Workflow purpose                                                                                             |
| MailerLite group assignment triggers email sequences for onboarding or upsell automations.                                      | https://www.mailerlite.com/a/Kr9Yplim6ZhV                                                                   |
| Gumroad API setup requires creating an application and generating an access token.                                               | https://gumroad.com                                                                                          |
| Google Sheets API setup requires OAuth2 credentials and enabling the Sheets API in Google Cloud Console.                         | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                               |
| Join the support Discord for help with the workflow: https://discord.gg/eBZH4WHCqd                                               | Support channel                                                                                              |
| MailerLite API documentation for groups: https://developers.mailerlite.com/docs/groups.html                                      | API reference for assigning subscribers to groups                                                           |
| Use MailerLite automation triggered by group membership to send multi-email nurture sequences.                                  | https://www.mailerlite.com/a/Kr9Yplim6ZhV                                                                   |

---

This documentation fully describes the workflow structure, node configurations, and setup instructions to enable reproduction, modification, and troubleshooting by advanced users and AI agents alike.