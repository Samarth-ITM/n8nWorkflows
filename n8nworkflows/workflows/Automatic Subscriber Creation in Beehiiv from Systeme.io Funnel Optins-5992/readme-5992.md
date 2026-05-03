Automatic Subscriber Creation in Beehiiv from Systeme.io Funnel Optins

https://n8nworkflows.xyz/workflows/automatic-subscriber-creation-in-beehiiv-from-systeme-io-funnel-optins-5992


# Automatic Subscriber Creation in Beehiiv from Systeme.io Funnel Optins

### 1. Workflow Overview

This workflow automates the creation of new subscribers in a Beehiiv publication whenever a new opt-in occurs in a Systeme.io sales funnel. It is designed for email marketing teams or automation specialists who want to synchronize Systeme.io funnel opt-in data with their Beehiiv subscriber list, including tracking UTM parameters and custom fields for subscriber names.

The workflow logically breaks down into the following blocks:

- **1.1 Input Reception:** Captures new opt-in events from Systeme.io via a secured webhook.
- **1.2 Configuration Setup:** Loads static configuration parameters such as Beehiiv publication ID and custom field names.
- **1.3 Data Cleansing and Extraction:** Normalizes and extracts relevant subscriber data and tracking parameters from the incoming webhook payload.
- **1.4 Beehiiv API Integration:** Sends a request to create a new subscriber in Beehiiv with the cleaned data.
- **1.5 Error Handling and Notification:** Checks for unsuccessful subscriber creation and sends an email alert if an error occurs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new opt-in POST requests from Systeme.io at a dedicated webhook endpoint. It restricts access to specific Systeme.io IP addresses for security.

- **Nodes Involved:**  
  - On New Systeme.io Optin

- **Node Details:**  
  - **On New Systeme.io Optin**  
    - Type: Webhook  
    - Role: Entry point for incoming opt-in data from Systeme.io funnel.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `funnel-level` (unique webhook URL segment)  
      - IP Whitelist: Includes three Systeme.io IP addresses (185.236.142.1, .2, .3) to accept requests only from those sources.  
    - Inputs: External HTTP POST requests from Systeme.io  
    - Outputs: Passes incoming webhook JSON to next node  
    - Edge Cases:  
      - Requests from unauthorized IPs are rejected.  
      - Missing or malformed webhook payloads may cause downstream expression failures.  

#### 2.2 Configuration Setup

- **Overview:**  
  Sets static and environment-specific parameters needed throughout the workflow, including Beehiiv publication ID, custom field names for first and last name, and email recipients for alerts.

- **Nodes Involved:**  
  - Configure Workflow

- **Node Details:**  
  - **Configure Workflow**  
    - Type: Set  
    - Role: Defines key configuration variables for workflow operation.  
    - Configuration:  
      - `beehiiv_publication_id`: Beehiiv publication identifier (example: "pub_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")  
      - `beehiiv_firstname_field_name`: Custom field name for first name in Beehiiv ("firstname")  
      - `beehiiv_lastname_field_name`: Custom field name for last name in Beehiiv ("lastname")  
      - `email_alert_recipients`: Comma-separated list of emails to receive error notifications ("recipient@example.com")  
    - Inputs: Receives webhook data from previous node  
    - Outputs: Passes configuration data merged with webhook data to next node  
    - Edge Cases:  
      - Missing or incorrect publication ID or field names will cause API errors later.  
      - Email alert recipients must be valid emails to receive notifications.

#### 2.3 Data Cleansing and Extraction

- **Overview:**  
  Extracts and normalizes subscriber email, names, referring URL, and UTM parameters from the incoming webhook data to prepare the payload for Beehiiv.

- **Nodes Involved:**  
  - Clean Data

- **Node Details:**  
  - **Clean Data**  
    - Type: Set (JSON mode)  
    - Role: Parse, clean, and structure webhook data into a simplified JSON object.  
    - Configuration:  
      - Extracts `email` from `body.data.contact.email`  
      - Extracts `first_name` from `body.data.contact.fields.first_name` or empty string if missing  
      - Extracts `last_name` from `body.data.contact.fields.surname` or empty string if missing  
      - Extracts `referring_site` by removing query parameters from `body.data.source_url`  
      - Extracts UTM parameters (`utm_source`, `utm_medium`, `utm_campaign`) from URL query parameters with regex, defaulting to empty strings if absent  
    - Uses expressions to safely handle missing fields and regex extraction.  
    - Inputs: Receives webhook and configuration data  
    - Outputs: Passes cleaned subscriber data to Beehiiv API request node  
    - Edge Cases:  
      - Missing or malformed URL in `source_url` may lead to empty or incorrect UTM/referring_site values.  
      - Missing name fields handled gracefully by defaulting to empty strings.

#### 2.4 Beehiiv API Integration

- **Overview:**  
  Sends a POST request to the Beehiiv API to create a subscriber with the extracted data, including UTM tags and custom fields for names.

- **Nodes Involved:**  
  - Create New Beehiiv Subscriber

- **Node Details:**  
  - **Create New Beehiiv Subscriber**  
    - Type: HTTP Request  
    - Role: Calls Beehiiv API to add a new subscriber to the specified publication.  
    - Configuration:  
      - URL: `https://api.beehiiv.com/v2/publications/{{ publication_id }}/subscriptions` dynamically uses configured Beehiiv publication ID  
      - Method: POST  
      - Authentication: HTTP Bearer Token (Beehiiv API key stored as credential)  
      - Headers: `Content-Type: application/json`  
      - Body: JSON payload includes email, UTM source/medium/campaign, referring site, and custom fields for first and last name using configured field names.  
      - Response: Configured to not throw errors on non-2xx status to allow manual checking downstream.  
    - Inputs: Cleaned subscriber data and configuration  
    - Outputs: API response JSON passed to conditional node  
    - Edge Cases:  
      - API key missing or invalid causes authorization errors.  
      - Network timeouts or rate limits may cause failures.  
      - Incorrect publication ID or field names cause 4xx errors.  

#### 2.5 Error Handling and Notification

- **Overview:**  
  Checks if the Beehiiv subscriber creation was successful (HTTP 200 or 201). If not, sends an email alert to notify about the failure.

- **Nodes Involved:**  
  - Subscriber Created?  
  - Send Email Alert (Beehiiv API error)

- **Node Details:**  
  - **Subscriber Created?**  
    - Type: If  
    - Role: Evaluates the HTTP status code of the Beehiiv API response.  
    - Configuration: Condition checks if statusCode is NOT 200 AND NOT 201, meaning failure.  
    - Inputs: API response from previous node  
    - Outputs:  
      - True branch: error path to email alert node  
      - False branch: workflow ends normally  
    - Edge Cases:  
      - Unexpected or malformed response could cause expression errors.  
  - **Send Email Alert (Beehiiv API error)**  
    - Type: Gmail node  
    - Role: Sends an alert email detailing the API error to configured recipients.  
    - Configuration:  
      - To: Configured alert email recipients  
      - Subject: "Systeme.io > Beehiiv Synchronization Error"  
      - Body: Includes subscriber email, API error status text, error code, and first error message from Beehiiv response.  
      - Uses Gmail OAuth2 credentials  
    - Inputs: Error branch of If node  
    - Outputs: None (terminal node)  
    - Edge Cases:  
      - Missing or invalid Gmail credentials cause email sending failure.  
      - Multiple recipients should be comma-separated.  
      - Long error messages may truncate in email clients.  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                                   | Input Node(s)                | Output Node(s)                       | Sticky Note                                                                                                                          |
|-------------------------------|--------------------|-------------------------------------------------|------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| On New Systeme.io Optin        | Webhook            | Receive new opt-in events from Systeme.io funnel | (start)                      | Configure Workflow                  | Automatically triggered with each new opt-in on your sales funnel. Open node for webhook URL needed to configure Systeme.io funnel.  |
| Configure Workflow             | Set                | Set static config: Beehiiv IDs, field names, alert emails | On New Systeme.io Optin       | Clean Data                         | Configure these 4 variables: beehiiv_publication_id, beehiiv_firstname_field_name, beehiiv_lastname_field_name, email_alert_recipients |
| Clean Data                    | Set                | Extract and normalize subscriber and tracking data | Configure Workflow            | Create New Beehiiv Subscriber      |                                                                                                                                      |
| Create New Beehiiv Subscriber  | HTTP Request       | Create subscriber via Beehiiv API                 | Clean Data                   | Subscriber Created?                 | Connect your Beehiiv account (API key). See https://www.beehiiv.com/support/article/13091918395799-how-to-access-your-publication-id-or-api-keys |
| Subscriber Created?            | If                 | Check if subscriber creation succeeded (200/201) | Create New Beehiiv Subscriber | Send Email Alert (if error)        |                                                                                                                                      |
| Send Email Alert (Beehiiv API error) | Gmail          | Notify via email if API call failed               | Subscriber Created? (error)  | (end)                             | Connect your Gmail account.                                                                                                         |
| Sticky Note4                  | Sticky Note        | Documentation and overview of workflow            | (none)                      | (none)                            | Detailed description, setup instructions, and benefits. See https://n8n.io/creators/belmehel/                                         |
| Sticky Note5                  | Sticky Note        | Notes about webhook trigger and IP whitelist      | (none)                      | (none)                            | Automatically triggered with each new opt-in on your sales funnel. Open node for webhook URL.                                         |
| Sticky Note6                  | Sticky Note        | Configuration instructions for Set node           | (none)                      | (none)                            | Set the values to these 4 variables: beehiiv_publication_id, beehiiv_firstname_field_name, beehiiv_lastname_field_name, email_alert_recipients |
| Sticky Note7                  | Sticky Note        | Beehiiv API connection instructions                | (none)                      | (none)                            | Connect your Beehiiv account (API key). See https://www.beehiiv.com/support/article/13091918395799-how-to-access-your-publication-id-or-api-keys |
| Sticky Note8                  | Sticky Note        | Gmail connection instructions                       | (none)                      | (none)                            | Connect your Gmail account.                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "On New Systeme.io Optin"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `funnel-level` (or your desired unique path)  
   - IP Whitelist: Add Systeme.io IPs: `185.236.142.1, 185.236.142.2, 185.236.142.3`  
   - Purpose: Receive opt-in webhook from Systeme.io funnel  
   - Save and activate to get the webhook URL for Systeme.io configuration.

2. **Create Set Node: "Configure Workflow"**  
   - Type: Set  
   - Add fields:  
     - `beehiiv_publication_id` (string): Your Beehiiv publication ID (e.g., "pub_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")  
     - `beehiiv_firstname_field_name` (string): Custom field name for first name in Beehiiv (default "firstname")  
     - `beehiiv_lastname_field_name` (string): Custom field name for last name in Beehiiv (default "lastname")  
     - `email_alert_recipients` (string): One or more email addresses separated by commas for error alerts  
   - Connect input from "On New Systeme.io Optin".

3. **Create Set Node: "Clean Data"**  
   - Type: Set (JSON mode)  
   - Use expressions to extract and normalize data:  
     - `email`: `{{$json["body"]["data"]["contact"]["email"]}}`  
     - `first_name`: `{{$json["body"]["data"]["contact"]["fields"]["first_name"] ?? ""}}`  
     - `last_name`: `{{$json["body"]["data"]["contact"]["fields"]["surname"] ?? ""}}`  
     - `referring_site`: Extract from `body.data.source_url` by stripping query params (regex or JS replace)  
     - `utm_source`: Extract `utm_source` param from `source_url` using regex  
     - `utm_medium`: Extract `utm_medium` param similarly  
     - `utm_campaign`: Extract `utm_campaign` param similarly  
   - Connect input from "Configure Workflow".

4. **Create HTTP Request Node: "Create New Beehiiv Subscriber"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.beehiiv.com/v2/publications/{{ $json["beehiiv_publication_id"] }}/subscriptions`  
   - Authentication: HTTP Bearer Token (set your Beehiiv API key in credentials)  
   - Headers: `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "email": "{{ $json.email }}",
       "utm_source": "{{ $json.utm_source }}",
       "utm_medium": "{{ $json.utm_medium }}",
       "utm_campaign": "{{ $json.utm_campaign }}",
       "referring_site": "{{ $json.referring_site }}",
       "custom_fields": [
         {
           "name": "{{ $json.beehiiv_firstname_field_name }}",
           "value": "{{ $json.first_name }}"
         },
         {
           "name": "{{ $json.beehiiv_lastname_field_name }}",
           "value": "{{ $json.last_name }}"
         }
       ]
     }
     ```  
   - Options: Configure to never throw error on HTTP errors to allow manual handling.  
   - Connect input from "Clean Data".

5. **Create If Node: "Subscriber Created?"**  
   - Type: If  
   - Condition: Check if `$json.statusCode` is NOT 200 AND NOT 201  
     - Use an AND combinator with two conditions:  
       - `$json.statusCode != 200`  
       - `$json.statusCode != 201`  
   - Connect input from "Create New Beehiiv Subscriber".

6. **Create Gmail Node: "Send Email Alert (Beehiiv API error)"**  
   - Type: Gmail  
   - Connect input from the "true" (error) branch of "Subscriber Created?"  
   - Configure Gmail OAuth2 credentials for sending email  
   - Parameters:  
     - To: `{{$json.email_alert_recipients}}` from configuration  
     - Subject: "Systeme.io > Beehiiv Synchronization Error"  
     - Message (plain text):  
       ```
       An error occurred while calling the Beehiiv API and the workflow has stopped.

       Subscriber affected: {{ $json.email }}
       Error status: {{ $json.body.statusText }} ({{ $json.body.status }})
       Error message: {{ $json.body.errors[0].message }}
       ```  
   - No output connection (terminal node).

7. **Connect all nodes in sequence:**  
   - On New Systeme.io Optin → Configure Workflow → Clean Data → Create New Beehiiv Subscriber → Subscriber Created?  
   - Subscriber Created? (true branch) → Send Email Alert (Beehiiv API error)  
   - Subscriber Created? (false branch) → end workflow

8. **Activate the workflow** and configure Systeme.io sales funnel webhook to the generated URL of "On New Systeme.io Optin".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow automatically creates a subscriber in Beehiiv when a new opt-in is registered in Systeme.io sales funnel. Integration is at the funnel level, not account level. For multiple funnels, duplicate or adapt workflow accordingly.                                                                                                                                                                                           | See sticky note inside workflow ("Sticky Note4")                                                                     |
| Configure your Systeme.io funnel to trigger a webhook after an opt-in. Use the webhook URL provided by the "On New Systeme.io Optin" node.                                                                                                                                                                                                                                                                                             | https://help.systeme.io/article/144-how-to-create-and-trigger-a-webhook-after-an-opt-in-or-a-sale                    |
| Find your Beehiiv publication ID and API keys in your Beehiiv account dashboard to configure the workflow properly.                                                                                                                                                                                                                                                                                                                  | https://www.beehiiv.com/support/article/13091918395799-how-to-access-your-publication-id-or-api-keys                |
| Configure custom field names in Beehiiv for first and last names if you want to send them along with the subscriber data.                                                                                                                                                                                                                                                                                                            | https://www.beehiiv.com/support/article/7712894720023-using-custom-fields-with-your-subscribers                      |
| Connect your Beehiiv API key in the HTTP Request node for authentication. Connect your Gmail account with OAuth2 credentials in the Gmail node to send alert emails.                                                                                                                                                                                                                                                                 |                                                                                                                                                             |
| Benefits include automation and scaling of email marketing efforts, elimination of manual subscriber list updates, and focus on content creation rather than technical tasks.                                                                                                                                                                                                                                                        |                                                                                                                                                             |
| Author's other templates and workflows are available for further automation inspiration.                                                                                                                                                                                                                                                                                                                                             | https://n8n.io/creators/belmehel/                                                                                   |

---

**Disclaimer:** The text provided herein is extracted exclusively from an n8n automation workflow. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.