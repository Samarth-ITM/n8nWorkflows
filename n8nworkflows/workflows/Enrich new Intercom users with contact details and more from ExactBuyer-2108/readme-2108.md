Enrich new Intercom users with contact details and more from ExactBuyer

https://n8nworkflows.xyz/workflows/enrich-new-intercom-users-with-contact-details-and-more-from-exactbuyer-2108


# Enrich new Intercom users with contact details and more from ExactBuyer

### 1. Workflow Overview

This workflow is designed to automatically enrich new user contacts created in Intercom with additional contact and location details obtained from ExactBuyer’s enrichment API. Its primary use case is to improve the quality and completeness of Intercom user profiles by supplementing them with verified phone numbers, emails, social profiles, and geographic data.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives webhook notifications from Intercom when a new user contact is created.
- **1.2 Event Filtering:** Determines if the webhook event corresponds to a "contact.user.created" event.
- **1.3 Data Extraction:** Extracts key identifiers (user ID and email) from the incoming Intercom event payload.
- **1.4 Data Enrichment:** Queries the ExactBuyer API using the extracted email to retrieve enriched contact and location information.
- **1.5 Data Processing:** Formats and massages the enrichment data, particularly social profiles and location fields, to match Intercom’s expected structure.
- **1.6 Data Update:** Updates the Intercom user profile with the enriched data via Intercom’s REST API.
- **1.7 Error/Edge Handling:** Handles cases where a user cannot be enriched (e.g., not found in ExactBuyer) or when an event is not a user creation event.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming webhook POST requests from Intercom. It is triggered whenever Intercom sends a user-related event as configured on the Intercom platform.

- **Nodes Involved:**  
  - On Webhook event from Intercom

- **Node Details:**  
  - **Node Name:** On Webhook event from Intercom  
  - **Type:** Webhook node  
  - **Configuration:**  
    - HTTP Method: POST  
    - Path: A unique webhook path `"11e21ebc-27ef-49b5-8c77-648faf3e86e0"` (configured in Intercom to send user events here)  
    - No additional options  
  - **Expressions/Variables:** Expects JSON body containing Intercom event data, including event type under `$json.body.topic` and user data under `$json.body.data.item`  
  - **Inputs:** External HTTP POST requests from Intercom  
  - **Outputs:** Passes webhook payload downstream to event filtering  
  - **Potential Failures:**  
    - Webhook misconfiguration in Intercom (wrong URL or event types not enabled)  
    - Payload format changes in Intercom API  
  - **Sub-workflow Reference:** None

---

#### 2.2 Event Filtering

- **Overview:**  
  Filters incoming webhook events to process only those where a new user contact is created (`contact.user.created`). Other events are routed separately.

- **Nodes Involved:**  
  - On user created  
  - Other event (NoOp)  

- **Node Details:**  
  - **Node Name:** On user created  
  - **Type:** Switch node  
  - **Configuration:**  
    - Condition: `$json.body.topic` equals `"contact.user.created"`  
    - Outputs:  
      - `"user created"` output for matching events  
      - `"extra"` fallback output for all other events  
  - **Inputs:** From webhook node  
  - **Outputs:**  
    - `"user created"`: flows to data extraction  
    - `"extra"`: flows to Other event node (NoOp)  
  - **Potential Failures:**  
    - Expression evaluation errors if `$json.body.topic` is missing or malformed  
    - Unhandled events simply routed to NoOp (safe)  
  - **Sub-workflow Reference:** None  

  - **Node Name:** Other event  
  - **Type:** NoOp (no operation)  
  - **Purpose:** Placeholder sink for unprocessed events, ensuring workflow stability  
  - **Inputs:** From Switch node `'extra'` output  
  - **Outputs:** None  

---

#### 2.3 Data Extraction

- **Overview:**  
  Extracts the Intercom user ID and email from the event payload to prepare for enrichment querying.

- **Nodes Involved:**  
  - set key fields

- **Node Details:**  
  - **Node Name:** set key fields  
  - **Type:** Set node  
  - **Configuration:**  
    - Sets two string fields:  
      - `user_id` = `$json.body.data.item.id` (Intercom’s unique user ID)  
      - `email` = `$json.body.data.item.email` (user email)  
  - **Inputs:** From Switch node `"user created"` output  
  - **Outputs:** Passes extracted fields downstream to enrichment query  
  - **Potential Failures:**  
    - Missing or null `email` or `user_id` in payload could break downstream calls  
  - **Sub-workflow Reference:** None

---

#### 2.4 Data Enrichment

- **Overview:**  
  Uses the ExactBuyer API to enrich the user profile based on the extracted email. The API returns contact details including work email, personal email, phone numbers, social profiles, and location data.

- **Nodes Involved:**  
  - Enrich user from ExactBuyer  

- **Node Details:**  
  - **Node Name:** Enrich user from ExactBuyer  
  - **Type:** HTTP Request node  
  - **Configuration:**  
    - HTTP Method: GET (default for query)  
    - URL: `https://api.exactbuyer.com/v1/enrich`  
    - Query Parameters:  
      - `email` set to the extracted email (`{{$json.email}}`)  
      - `required` set to `"work_email,personal_email,email"` (requesting these fields explicitly)  
    - Authentication: HTTP Header Auth using ExactBuyer API key credential  
    - On error: Continue with error output (to allow graceful failure handling)  
  - **Inputs:** From set key fields node  
  - **Outputs:** Two possible outputs:  
    - Success output passing enrichment data to data processing  
    - Error/empty output routed to "Could not find user" node for logging or alternate handling  
  - **Potential Failures:**  
    - Invalid or expired ExactBuyer API key causing 401 Unauthorized  
    - API rate limits or downtime causing timeouts or 5xx errors  
    - Email not found in ExactBuyer database resulting in empty or invalid response  
  - **Sub-workflow Reference:** None

---

#### 2.5 Data Processing

- **Overview:**  
  Processes the ExactBuyer enrichment response to construct properly formatted social profiles and location objects compatible with Intercom’s update format.

- **Nodes Involved:**  
  - massage data  

- **Node Details:**  
  - **Node Name:** massage data  
  - **Type:** Code node (JavaScript)  
  - **Configuration:**  
    - Runs once for each item  
    - JavaScript code:  
      - Maps over `result.social_profiles` from ExactBuyer to create array of objects with keys: `type: 'social_profile'`, `name` (network), and `url`  
      - Extracts `country`, `city`, and `region` from `result.location` into a new `location` object  
      - Returns modified item with added `social_profiles` and `location` fields  
  - **Inputs:** From successful output of enrichment node  
  - **Outputs:** Passes massaged data to Intercom update node  
  - **Potential Failures:**  
    - Null or undefined fields in enrichment response causing runtime JS errors  
    - Unexpected social profiles data structure causing map failures  
  - **Sub-workflow Reference:** None

---

#### 2.6 Data Update

- **Overview:**  
  Updates the Intercom user contact with enriched profile details including email, name, phone, avatar, social profiles, and location.

- **Nodes Involved:**  
  - Update data in Intercom  

- **Node Details:**  
  - **Node Name:** Update data in Intercom  
  - **Type:** HTTP Request node  
  - **Configuration:**  
    - HTTP Method: PUT  
    - URL: `https://api.intercom.io/contacts/{{user_id}}` (user_id from set key fields node)  
    - Headers:  
      - `Intercom-Version: 2.10` (required API version header)  
      - Authentication: HTTP Header Auth with Intercom API key credential  
    - Body: JSON with fields:  
      - `email` (work email from enrichment + full name concatenation — note: concatenation may be accidental or incorrect)  
      - `name` (full name from enrichment)  
      - `phone` (first phone number in E164 format)  
      - `avatar` (profile picture URL)  
      - `social_profiles` (array created in massage data)  
      - `location` (location object created in massage data)  
    - Sends both body and headers  
  - **Inputs:** From massage data node  
  - **Outputs:** None (final step)  
  - **Potential Failures:**  
    - Invalid or expired Intercom API key causing 401 errors  
    - Invalid data format causing API validation errors  
    - Network or timeout failures  
  - **Sub-workflow Reference:** None

---

#### 2.7 Error/Edge Handling

- **Overview:**  
  Handles cases where enrichment fails or user is not found, ensuring the workflow does not break silently.

- **Nodes Involved:**  
  - Could not find user  

- **Node Details:**  
  - **Node Name:** Could not find user  
  - **Type:** NoOp node  
  - **Purpose:** Placeholder sink for failed enrichment attempts or missing users  
  - **Inputs:** From error output of enrichment node  
  - **Outputs:** None  
  - **Potential Failures:** None (safe dead-end)

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                          |
|-----------------------------|---------------------|---------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------|
| On Webhook event from Intercom | Webhook             | Receives webhook from Intercom        | External HTTP POST             | On user created             | Setup webhook url in Intercom; Make sure `contact.user.created` event is enabled                                      |
| On user created             | Switch              | Filters for user created events       | On Webhook event from Intercom | set key fields; Other event |                                                                                                                      |
| set key fields              | Set                 | Extracts user_id and email from event | On user created               | Enrich user from ExactBuyer |                                                                                                                      |
| Enrich user from ExactBuyer | HTTP Request        | Calls ExactBuyer API for enrichment   | set key fields                | massage data; Could not find user | Add API key from ExactBuyer; Use email as identifier; See https://docs.exactbuyer.com/contact-enrichment/enrichment |
| massage data                | Code                | Formats enrichment data for Intercom  | Enrich user from ExactBuyer   | Update data in Intercom     |                                                                                                                      |
| Update data in Intercom     | HTTP Request        | Updates user profile in Intercom      | massage data                  | None                        | Set HTTP node and generic header API key; See https://developers.intercom.com/docs/build-an-integration/learn-more/authentication/ and https://developers.intercom.com/docs/references/rest-api/api.intercom.io/Contacts/UpdateContact/ |
| Could not find user         | NoOp                | Handles missing user/enrichment failure | Enrich user from ExactBuyer (error output) | None                    |                                                                                                                      |
| Other event                 | NoOp                | Handles all other non-user-created events | On user created (fallback)     | None                        |                                                                                                                      |
| Sticky Note                 | Sticky Note         | Documentation notes                   | None                         | None                        | ## On User created event in Intercom: 1. Setup webhook url in intercom; 2. Make sure `contact.user.created` is enabled |
| Sticky Note1                | Sticky Note         | Documentation notes                   | None                         | None                        | ## Enrich data from ExactBuyer: 1. Add api key from Exact buyer; 2. Use email as identifier to match user; API Guide https://docs.exactbuyer.com/contact-enrichment/enrichment |
| Sticky Note2                | Sticky Note         | Documentation notes                   | None                         | None                        | ## Update user in Intercom: 1. Set Http node and generic header API Key using this guide https://developers.intercom.com/docs/build-an-integration/learn-more/authentication/; 2. Update data in intercom using this guide https://developers.intercom.com/docs/references/rest-api/api.intercom.io/Contacts/UpdateContact/ |
| Sticky Note3                | Sticky Note         | Documentation notes                   | None                         | None                        | # Enrich new Intercom users with contact details from ExactBuyer; This workflow aims to enrich new contacts in Intercom. The more relevant the Intercom profile, the more useful it is. Once active, this n8n workflow will update the social profiles, contact data (phone, email) as well as location data from ExactBuyer. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `On Webhook event from Intercom`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `11e21ebc-27ef-49b5-8c77-648faf3e86e0` (or your custom path)  
   - This node is the entry point to receive Intercom events.

2. **Create Switch Node:**  
   - Name: `On user created`  
   - Type: Switch  
   - Add condition:  
     - Field: `$json.body.topic`  
     - Operation: Equals  
     - Value: `contact.user.created`  
   - Set fallback output to `"extra"`.

3. **Create NoOp Node for Other Events:**  
   - Name: `Other event`  
   - Type: NoOp  
   - Connect fallback `"extra"` output from the switch node to this node.

4. **Create Set Node to Extract Keys:**  
   - Name: `set key fields`  
   - Type: Set  
   - Add two fields:  
     - `user_id` (string) = `{{$json.body.data.item.id}}`  
     - `email` (string) = `{{$json.body.data.item.email}}`  
   - Connect `"user created"` output from switch node to this node.

5. **Create HTTP Request Node to Enrich from ExactBuyer:**  
   - Name: `Enrich user from ExactBuyer`  
   - Type: HTTP Request  
   - HTTP Method: GET (default)  
   - URL: `https://api.exactbuyer.com/v1/enrich`  
   - Query Parameters:  
     - `email`: `{{$json.email}}`  
     - `required`: `work_email,personal_email,email`  
   - Authentication: HTTP Header Auth (Add ExactBuyer API key credential)  
   - Error handling: On error continue (to handle failed enrichments gracefully)  
   - Connect from `set key fields`.

6. **Create Code Node to Massage Data:**  
   - Name: `massage data`  
   - Type: Code (JavaScript)  
   - Run once per item  
   - Code snippet:  
     ```javascript
     $input.item.json.social_profiles = $input.item.json.result.social_profiles.map(profile => ({
       type: 'social_profile',
       name: profile.network,
       url: profile.url,
     }));

     $input.item.json.location = {
       country: $input.item.json.result.location?.country,
       city: $input.item.json.result.location?.city,
       region: $input.item.json.result.location?.region,
     };

     return $input.item;
     ```  
   - Connect success output from ExactBuyer enrichment node to this node.

7. **Create HTTP Request Node to Update Data in Intercom:**  
   - Name: `Update data in Intercom`  
   - Type: HTTP Request  
   - HTTP Method: PUT  
   - URL: `https://api.intercom.io/contacts/{{$node['set key fields'].json.user_id}}`  
   - Headers:  
     - `Intercom-Version`: `2.10`  
   - Authentication: HTTP Header Auth (Add Intercom API key credential)  
   - Body (JSON):  
     ```json
     {
       "email": "={{ $json.result.current_work_email }} {{ $json.result.full_name }}",
       "name": "={{ $json.result.full_name }}",
       "phone": "={{ $json.result.phone_numbers?.[0]?.E164 }}",
       "avatar": "={{ $json.result.employment?.profile_pic_url }}",
       "social_profiles": "={{ $json.social_profiles }}",
       "location": "={{ $json.location }}"
     }
     ```  
   - Connect output of `massage data` node to this node.

8. **Create NoOp Node for Could Not Find User:**  
   - Name: `Could not find user`  
   - Type: NoOp  
   - Connect error output from `Enrich user from ExactBuyer` node to this node.

9. **Connect all nodes accordingly:**  
   - `On Webhook event from Intercom` → `On user created`  
   - `On user created`  
     - `"user created"` output → `set key fields` → `Enrich user from ExactBuyer`  
     - `"extra"` output → `Other event`  
   - `Enrich user from ExactBuyer`  
     - Success output → `massage data` → `Update data in Intercom`  
     - Error output → `Could not find user`

10. **Credentials Setup:**  
    - Configure and add credentials for:  
      - ExactBuyer API key (HTTP Header Auth)  
      - Intercom API key (HTTP Header Auth)  

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Add webhook URL in Intercom and ensure the `contact.user.created` event is enabled to trigger this workflow.              | Sticky Note on webhook node                                                                                                                   |
| Use ExactBuyer API guide for enriching contact details: https://docs.exactbuyer.com/contact-enrichment/enrichment          | Sticky Note near ExactBuyer enrichment node                                                                                                  |
| Set up Intercom HTTP Request node and API key authentication as per Intercom's official docs:                             | https://developers.intercom.com/docs/build-an-integration/learn-more/authentication/ and https://developers.intercom.com/docs/references/rest-api/api.intercom.io/Contacts/UpdateContact/ |
| This workflow improves Intercom contact data quality by adding social profiles, phone, email, avatar, and location info.  | General workflow description (Sticky Note3)                                                                                                  |

---

This documentation provides a full understanding of the workflow logic, node-by-node configuration, potential failure points, and instructions to reproduce or modify the workflow confidently.