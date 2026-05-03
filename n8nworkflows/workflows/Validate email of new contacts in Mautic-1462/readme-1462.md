Validate email of new contacts in Mautic

https://n8nworkflows.xyz/workflows/validate-email-of-new-contacts-in-mautic-1462


# Validate email of new contacts in Mautic

### 1. Workflow Overview

This workflow is designed to automatically validate the email addresses of new contacts created in Mautic. When a new contact is saved in Mautic, the workflow triggers, extracts contact details, validates the email address using OneSimpleAPI, and if the email is deemed suspicious (e.g., poor deliverability, invalid domain, disposable email), it sends an alert message to a specified Slack channel.

The workflow is logically divided into the following blocks:

- **1.1 Mautic Contact Trigger:** Detect new contacts created in Mautic.
- **1.2 Contact Data Extraction:** Extract detailed contact information from the triggered event.
- **1.3 Email Validation:** Check the extracted email address using OneSimpleAPI.
- **1.4 Suspicious Email Detection:** Evaluate the email validation results to decide if the email is suspicious.
- **1.5 Slack Notification:** Notify the relevant Slack channel about suspicious emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Mautic Contact Trigger

- **Overview:**  
  This block listens for new contact creation events in Mautic using a webhook trigger node. It initiates the workflow when a new contact is saved.

- **Nodes Involved:**  
  - On Contact Identified  
  - If is not new contact

- **Node Details:**

  - **On Contact Identified**  
    - Type: Mautic Trigger  
    - Role: Listens for `mautic.lead_post_save_new` event, which indicates a new contact creation in Mautic.  
    - Configuration: Uses OAuth2 credentials for Mautic authentication. The webhook is preconfigured to receive new contact events.  
    - Input: None (trigger node)  
    - Output: Emits JSON containing the new contact data under `mautic.lead_post_save_new`.  
    - Edge Cases:  
      - OAuth2 token expiration or invalid credentials can cause trigger failure.  
      - Mautic webhook misconfiguration may lead to missed events.

  - **If is not new contact**  
    - Type: If (conditional)  
    - Role: Checks if the incoming event data corresponds to a new contact or not by inspecting if `mautic.lead_post_save_new` is empty.  
    - Configuration: Condition checks if `{{$json["mautic.lead_post_save_new"]}}` is empty.  
    - Input: From "On Contact Identified" node  
    - Output:  
      - On False (not empty): proceeds to extract information.  
      - On True (empty): stops workflow (no further output).  
    - Edge Cases: If the event data structure changes, this condition might fail or misclassify events.

#### 2.2 Contact Data Extraction

- **Overview:**  
  This block extracts the relevant contact information from the nested JSON payload provided by Mautic for further processing.

- **Nodes Involved:**  
  - extract information

- **Node Details:**

  - **extract information**  
    - Type: Item Lists  
    - Role: Splits out the `mautic.lead_post_save_new` field into individual items for processing.  
    - Configuration: Field to split: `mautic.lead_post_save_new`  
    - Input: From "If is not new contact" (false branch)  
    - Output: Emits a simplified JSON structure representing the contact's data for easy access in subsequent nodes.  
    - Edge Cases: If the field is missing or empty, this node may emit empty output or errors.

#### 2.3 Email Validation

- **Overview:**  
  This block validates the extracted email address using the OneSimpleAPI service to assess deliverability, domain validity, and whether the email is disposable.

- **Nodes Involved:**  
  - validate email

- **Node Details:**

  - **validate email**  
    - Type: OneSimpleAPI (utility resource)  
    - Role: Calls OneSimpleAPI to validate the email address extracted from the contact details.  
    - Configuration: Email address is dynamically set using expression:  
      `{{$json["lead"]["fields"]["core"]["email"]["value"]}}`  
    - Credentials: OneSimpleAPI credentials required (API key)  
    - Input: From "extract information"  
    - Output: JSON containing validation results such as `deliverability`, `is_domain_valid`, and `is_email_disposable`.  
    - Edge Cases:  
      - API rate limits or invalid API key could cause failures.  
      - Invalid or missing email value could cause request failure.  
      - Network timeouts or service outages.

#### 2.4 Suspicious Email Detection

- **Overview:**  
  This block analyzes the response from OneSimpleAPI to determine if the email should be considered suspicious based on predefined criteria.

- **Nodes Involved:**  
  - If the email is suspicious

- **Node Details:**

  - **If the email is suspicious**  
    - Type: If (conditional)  
    - Role: Checks if any of these conditions are true:  
      - `deliverability` not equal to "GOOD"  
      - `is_domain_valid` is false  
      - `is_email_disposable` is true  
    - Configuration: Combines conditions with "any" logical operation (OR).  
    - Input: From "validate email"  
    - Output:  
      - True: Email is suspicious; proceed to Slack notification.  
      - False: Email is considered valid; workflow ends here.  
    - Edge Cases: If the API response structure changes, conditions may misfire. Also, borderline cases in deliverability scoring may require manual tuning.

#### 2.5 Slack Notification

- **Overview:**  
  This block sends an alert message to a specified Slack channel whenever a suspicious email is detected.

- **Nodes Involved:**  
  - Send to Slack

- **Node Details:**

  - **Send to Slack**  
    - Type: Slack  
    - Role: Sends a formatted alert message to a Slack channel to notify responsible teams about suspicious new contacts.  
    - Configuration:  
      - Channel: `#mautic-alerts` (configurable)  
      - Message text includes:  
        - Warning emoji and label  
        - Contact's full name (first and last name)  
        - Email address  
        - Link to contact in Mautic (using contact ID)  
        - Creator of the contact   
      - Uses expressions to pull data from "extract information" node, e.g.:  
        ```
        {{$node["extract information"].json["contact"]["fields"]["core"]["firstname"]["normalizedValue"]}}
        ```
    - Credentials: Slack API credentials (OAuth token or bot token) required  
    - Input: From "If the email is suspicious" (true branch)  
    - Output: Slack message posted; no further output.  
    - Edge Cases:  
      - Slack API rate limits or invalid/expired token can cause message failure.  
      - Incorrect channel configuration causes message to be sent to wrong place or fail.

---

### 3. Summary Table

| Node Name             | Node Type             | Functional Role                     | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                      |
|-----------------------|-----------------------|-----------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| On Contact Identified  | Mautic Trigger        | Trigger on new Mautic contact     | None                  | If is not new contact  |                                                                                                 |
| If is not new contact  | If                    | Check if contact is new            | On Contact Identified  | extract information    |                                                                                                 |
| extract information    | Item Lists            | Extract contact details            | If is not new contact  | validate email         |                                                                                                 |
| validate email        | OneSimpleAPI           | Validate contact email via API     | extract information    | If the email is suspicious |                                                                                                 |
| If the email is suspicious | If                 | Detect suspicious email            | validate email         | Send to Slack          | IF deliverability is not good OR Domain is not valid OR Email is Disposable                      |
| Send to Slack          | Slack                 | Send alert message for suspicious email | If the email is suspicious | None                  |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Mautic Trigger node**  
   - Set node name: `On Contact Identified`  
   - Set event to listen: `mautic.lead_post_save_new`  
   - Use OAuth2 credentials for Mautic (create/set if not existing)  
   - Configure webhook (automatic in n8n)  

3. **Add an If node**  
   - Name: `If is not new contact`  
   - Condition: Check if `{{$json["mautic.lead_post_save_new"]}}` is empty (string operation: "isEmpty")  
   - Connect `On Contact Identified` → `If is not new contact`

4. **Add an Item Lists node**  
   - Name: `extract information`  
   - Field to split out: `mautic.lead_post_save_new`  
   - Connect the false output (new contact detected) of `If is not new contact` to `extract information`

5. **Add a OneSimpleAPI node**  
   - Name: `validate email`  
   - Resource: `utility`  
   - Set `emailAddress` parameter to expression:  
     `{{$json["lead"]["fields"]["core"]["email"]["value"]}}`  
   - Provide OneSimpleAPI credentials (API key)  
   - Connect `extract information` → `validate email`

6. **Add another If node**  
   - Name: `If the email is suspicious`  
   - Conditions (combine with OR / "any"):  
     - String: `{{$json["deliverability"]}}` not equal to `"GOOD"`  
     - Boolean: `{{$json["is_domain_valid"]}}` is false  
     - Boolean: `{{$json["is_email_disposable"]}}` is true  
   - Connect `validate email` → `If the email is suspicious`

7. **Add a Slack node**  
   - Name: `Send to Slack`  
   - Channel: select or enter Slack channel (e.g. `#mautic-alerts`)  
   - Text: Use the following message template with expressions:  
     ```
     =:warning: New Contact with Suspicious Email :warning:
     *Name: * {{$node["extract information"].json["contact"]["fields"]["core"]["firstname"]["normalizedValue"]}} {{$node["extract information"].json["contact"]["fields"]["core"]["lastname"]["normalizedValue"]}}
     *Email: * {{$node["extract information"].json["contact"]["fields"]["core"]["email"]["normalizedValue"]}}
     *Link: * https://mautic.my.domain.com/s/contacts/view/{{$node["extract information"].json["contact"]["id"]}}
     *Creator: * {{$node["extract information"].json["contact"]["createdByUser"]}}
     ```  
   - Provide Slack credentials (OAuth or Bot token)  
   - Connect the true output of `If the email is suspicious` → `Send to Slack`

8. **Deactivate or end workflow on all other branches where no suspicious email is detected.**

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                           |
|------------------------------------------------------------------------------------------------|------------------------------------------|
| To configure this workflow, provide valid credentials for Mautic (OAuth2), OneSimpleAPI (API key), and Slack (OAuth/Bot Token). | Configuration instructions                |
| The Slack channel used for alerts should be set according to your team's notification preferences. | Slack channel configuration               |
| For detailed OneSimpleAPI documentation and examples, visit: https://onesimpleapi.com/docs    | OneSimpleAPI docs                         |
| Mautic webhook setup must allow `mautic.lead_post_save_new` event to trigger the workflow.     | Mautic webhook events documentation      |
| Slack API token must have permission to post messages in the selected channel.                  | Slack app permissions documentation       |

---

This structured documentation fully describes the workflow "Validate email of new contacts in Mautic," enabling advanced users and automation agents to understand, reproduce, and maintain it effectively.