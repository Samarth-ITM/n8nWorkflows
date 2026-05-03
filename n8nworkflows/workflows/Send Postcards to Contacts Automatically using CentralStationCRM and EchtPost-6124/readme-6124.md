Send Postcards to Contacts Automatically using CentralStationCRM and EchtPost

https://n8nworkflows.xyz/workflows/send-postcards-to-contacts-automatically-using-centralstationcrm-and-echtpost-6124


# Send Postcards to Contacts Automatically using CentralStationCRM and EchtPost

### 1. Workflow Overview

This workflow automates the sending of postcards to contacts newly updated in the CentralStationCRM system, using the EchtPost postcard sending service. It is designed for teams using CentralStationCRM to trigger postcard mailings when a contact's data is updated with a specific tag.

The logical blocks are:

- **1.1 Input Reception:** Receives webhook events from CentralStationCRM when a person record is updated.
- **1.2 Data Preparation:** Extracts and formats the necessary contact data fields required by EchtPost.
- **1.3 Tag Filtering:** Checks if the contact is tagged with "EchtPost" to determine whether to send a postcard.
- **1.4 Postcard Sending:** If tagged, sends a postcard via the EchtPost API.
- **1.5 No-Op Handling:** If not tagged, performs no action to gracefully end the workflow.

Supporting elements include detailed sticky notes with instructions for setup and configuration relevant to each step.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens to CentralStationCRM webhook POST requests triggered by updates to person records.

- **Nodes Involved:**  
  - Person is updated in CentralStationCRM (Webhook)

- **Node Details:**

  - **Person is updated in CentralStationCRM**  
    - Type: Webhook  
    - Role: Entry trigger node that waits for HTTP POST requests from CentralStationCRM webhook on person updates.  
    - Configuration:  
      - HTTP Method: POST  
      - Webhook Path: Unique UUID path defined automatically by n8n.  
    - Inputs: None (trigger node)  
    - Outputs: Sends webhook JSON payload to next node.  
    - Edge Cases:  
      - Network connectivity issues may cause missed triggers.  
      - Payload format changes from CentralStationCRM could cause parsing issues.  
      - Authentication is none, so webhook URL should be kept secret.  
    - Notes: This node requires the webhook URL to be registered in CentralStationCRM with monitored object type “Person” and event “Update”. The nested objects tags, addrs, companies must be included.

#### 2.2 Data Preparation

- **Overview:**  
  Extracts and passes only required fields from the webhook payload that are relevant for creating a postcard via EchtPost.

- **Nodes Involved:**  
  - Set fields needed for EchtPost

- **Node Details:**

  - **Set fields needed for EchtPost**  
    - Type: Set  
    - Role: Filters and maps selected fields from the incoming JSON to a simplified structure for postcard sending.  
    - Configuration:  
      - Includes fields:  
        - `body.record.companies[0].name` (Company name)  
        - `body.record.addrs[0].street`  
        - `body.record.addrs[0].zip`  
        - `body.record.addrs[0].city`  
        - `body.record.addrs[0].country_code`  
        - `body.record.taggings` (Tags array)  
      - Keeps other fields by default.  
    - Inputs: From webhook node.  
    - Outputs: Simplified JSON with contact address and tags for next node.  
    - Edge Cases:  
      - Missing address or company data could result in incomplete postcard data.  
      - Empty or null taggings array will affect filtering later.  
    - Notes: Ensures only relevant data is passed forward to reduce complexity.

#### 2.3 Tag Filtering

- **Overview:**  
  Checks if the "EchtPost" tag exists within the tags array of the contact to decide if a postcard should be sent.

- **Nodes Involved:**  
  - Is "EchtPost" included in "taggings"?

- **Node Details:**

  - **Is "EchtPost" included in "taggings"?**  
    - Type: If  
    - Role: Conditional node that routes workflow depending on presence of "EchtPost" in taggings.  
    - Configuration:  
      - Condition: String contains check on `taggings` field, case sensitive, looking for substring `"EchtPost"`.  
    - Inputs: From Set node.  
    - Outputs:  
      - True (tag found): proceed to sending postcard.  
      - False (tag not found): proceed to no-op node.  
    - Edge Cases:  
      - Tag names are case sensitive; "echtpost" (lowercase) will not match.  
      - Missing or malformed tags field may cause false negatives.  
    - Notes: This node effectively filters contacts to those that require postcards.

#### 2.4 Postcard Sending

- **Overview:**  
  If the tag "EchtPost" is found, this block sends a postcard using the EchtPost REST API with provided contact details.

- **Nodes Involved:**  
  - Send postcard with EchtPost

- **Node Details:**

  - **Send postcard with EchtPost**  
    - Type: HTTP Request  
    - Role: Sends a POST request to EchtPost API to create and deliver a postcard.  
    - Configuration:  
      - Method: POST  
      - URL: https://api.echtpost.de/v1/cards  
      - Authentication: None (API key included in JSON body)  
      - Body Content Type: JSON  
      - Body: JSON object including:  
        - `"apikey"`: Placeholder for user’s EchtPost API key (must be replaced).  
        - `"card"` object with:  
          - `"template_id"`: Placeholder for EchtPost template ID.  
          - `"deliver_at"`: Delivery time set as "3-days-from-now".  
          - `"notification_type"`: "before_send".  
          - `"notification_email"`: User email for notifications (placeholder).  
          - `"contacts_attributes"`: Array with one contact object mapping company name, street, zip, city, country_code from previous node JSON using expressions.  
    - Inputs: From If node (True branch).  
    - Outputs: Response from EchtPost API.  
    - Edge Cases:  
      - Missing or invalid API key will cause authentication errors.  
      - Invalid or expired template ID will cause API errors.  
      - Network or HTTP errors could cause request failure.  
      - Address fields missing or malformed may cause delivery issues.  
    - Notes: User must fill in API key, template ID, and notification email before activating workflow.

#### 2.5 No-Op Handling

- **Overview:**  
  Gracefully ends the workflow without action if the contact does not have the "EchtPost" tag.

- **Nodes Involved:**  
  - Do nothing

- **Node Details:**

  - **Do nothing**  
    - Type: NoOp (No Operation)  
    - Role: Acts as a sink for the False branch of the If node, explicitly doing nothing.  
    - Configuration: None.  
    - Inputs: From If node (False branch).  
    - Outputs: None.  
    - Edge Cases: None.  
    - Notes: Useful to clearly document workflow logic.

---

### 3. Summary Table

| Node Name                         | Node Type         | Functional Role                           | Input Node(s)                       | Output Node(s)                   | Sticky Note                                                                                                                                    |
|----------------------------------|-------------------|-----------------------------------------|-----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1                     | Sticky Note       | Overview, general instructions           | —                                 | —                               | ![CSCRM Logo](https://s3.42he.com/cscrm-marketing-page-production/Logo_Central_Station_CRM_0dd02e23d2.jpeg) Workflow overview and instructions. |
| Sticky Note                      | Sticky Note       | Instructions to set up CentralStationCRM webhook | —                                 | —                               | Setup instructions with screenshots for webhook in CentralStationCRM.                                                                           |
| Sticky Note2                    | Sticky Note       | Setup instructions for n8n webhook trigger | —                                 | —                               | Instructions for configuring the n8n webhook node and testing.                                                                                  |
| Sticky Note4                    | Sticky Note       | Info to obtain webhook URL from n8n      | —                                 | —                               | How to get webhook URL from n8n node.                                                                                                          |
| Sticky Note3                    | Sticky Note       | Tool references                          | —                                 | —                               | Mentions CentralStationCRM and EchtPost websites.                                                                                              |
| Sticky Note5                    | Sticky Note       | Instructions to get EchtPost API key and template ID | —                                 | —                               | Stepwise guide with images to obtain API key and template ID from EchtPost.                                                                     |
| Sticky Note7                    | Sticky Note       | Instructions to configure EchtPost API node | —                                 | —                               | Detailed JSON body instructions to configure the HTTP request node for EchtPost.                                                               |
| Person is updated in CentralStationCRM | Webhook           | Receive updates from CentralStationCRM   | —                                 | Set fields needed for EchtPost   |                                                                                                                                                 |
| Set fields needed for EchtPost  | Set               | Filter and map data for postcard sending | Person is updated in CentralStationCRM | Is "EchtPost" included in "taggings"? |                                                                                                                                                 |
| Is "EchtPost" included in "taggings"? | If                | Check if contact has "EchtPost" tag       | Set fields needed for EchtPost    | Send postcard with EchtPost; Do nothing |                                                                                                                                                 |
| Send postcard with EchtPost     | HTTP Request      | Send postcard via EchtPost API            | Is "EchtPost" included in "taggings"? (True) | —                               |                                                                                                                                                 |
| Do nothing                     | NoOp              | No action if tag not found                 | Is "EchtPost" included in "taggings"? (False) | —                               |                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node named `Person is updated in CentralStationCRM`.  
   - Set HTTP Method to **POST**.  
   - Accept default webhook path or generate a unique one.  
   - Save and copy the webhook URL.

2. **Register Webhook in CentralStationCRM**  
   - In CentralStationCRM, go to Account Settings → Webhooks.  
   - Create a new webhook with:  
     - Password (set as desired).  
     - Monitored object type: **Person**.  
     - Monitored event: **Update**.  
     - Endpoint URL: Paste the n8n webhook URL.  
     - Nested objects to include: `tags`, `addrs`, `companies`.  
   - Set webhook status to active and save.

3. **Add Set Node**  
   - Add a **Set** node named `Set fields needed for EchtPost`.  
   - Configure to include only the following fields from incoming data:  
     - `body.record.companies[0].name`  
     - `body.record.addrs[0].street`  
     - `body.record.addrs[0].zip`  
     - `body.record.addrs[0].city`  
     - `body.record.addrs[0].country_code`  
     - `body.record.taggings`  
   - Keep other fields if necessary.  
   - Connect input from webhook node.

4. **Add If Node for Tag Filtering**  
   - Add an **If** node named `Is "EchtPost" included in "taggings"?`.  
   - Condition: String Contains  
     - Left Value: Expression `{{$json["taggings"]}}`  
     - Right Value: `"EchtPost"` (case sensitive)  
   - Connect input from Set node.

5. **Add HTTP Request Node for Sending Postcard**  
   - Add an **HTTP Request** node named `Send postcard with EchtPost`.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.echtpost.de/v1/cards`  
     - Authentication: None (API key in JSON body)  
     - Send Body: true, Body Content Type: JSON  
     - Request Body (expression mode):  
       ```json
       {
         "apikey": "YOUR-ECHTPOST-API-KEY",
         "card": {
           "template_id": "YOUR-TEMPLATE-ID",
           "deliver_at": "3-days-from-now",
           "notification_type": "before_send",
           "notification_email": "YOUR-NOTIFICATION-EMAIL",
           "contacts_attributes": [
             {
               "company_name": "{{ $json.body.record.companies[0].name }}",
               "street": "{{ $json.body.record.addrs[0].street }}",
               "zip": "{{ $json.body.record.addrs[0].zip }}",
               "city": "{{ $json.body.record.addrs[0].city }}",
               "country_code": "{{ $json.body.record.addrs[0].country_code }}"
             }
           ]
         }
       }
       ```  
     - Replace placeholders with actual API key, template ID, and notification email.  
   - Connect input from If node (True branch).

6. **Add NoOp Node**  
   - Add a **NoOp** node named `Do nothing`.  
   - Connect input from If node (False branch).

7. **Finalize Connections**  
   - Connect `Person is updated in CentralStationCRM` → `Set fields needed for EchtPost` → `Is "EchtPost" included in "taggings"?`  
   - Connect If node True output → `Send postcard with EchtPost`  
   - Connect If node False output → `Do nothing`

8. **Activate Workflow**  
   - Double-check all configurations and expressions.  
   - Save the workflow.  
   - Set workflow status to **Active**.

9. **Testing**  
   - In CentralStationCRM, tag a person with "EchtPost" and update the record.  
   - Observe webhook trigger and postcard sending.  
   - Monitor EchtPost for postcard delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| CentralStationCRM is a simple and intuitive CRM for small teams.                                                               | https://centralstationcrm.de                                                                                               |
| EchtPost allows sending postcards online easily with API support.                                                              | https://echtpost.de                                                                                                        |
| To get EchtPost API key: Login to EchtPost, access settings via cogwheel, create API key, and copy it.                          | See sticky note with image: https://s3.42he.com/cscrm-marketing-page-production/Echt_Post_API_Schluessel_24d1c6be44.png     |
| To get EchtPost template ID: Click on a postcard template and copy the number after `?template_id=` in the URL.                 | See sticky note with image: https://s3.42he.com/cscrm-marketing-page-production/Echt_Post_Vorlagen_1592063ee7.png            |
| For detailed configuration of the EchtPost HTTP Request node, use the example JSON body and make sure to switch to Expression. | Provided in sticky note7 within the workflow.                                                                              |
| Important: Keep the webhook URL secret as it has no authentication configured.                                                  | Security best practice for webhook URLs.                                                                                   |
| Tag matching in the If node is case sensitive; ensure tags in CentralStationCRM match exactly "EchtPost".                       | Potential source of logic errors if casing differs.                                                                         |

---

**Disclaimer:** The text above is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.