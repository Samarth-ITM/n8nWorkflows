Verify mailing address deliverability of new contacts in HighLevel Using Lob

https://n8nworkflows.xyz/workflows/verify-mailing-address-deliverability-of-new-contacts-in-highlevel-using-lob-2171


# Verify mailing address deliverability of new contacts in HighLevel Using Lob

### 1. Workflow Overview

This workflow is designed to verify the deliverability of mailing addresses for new contacts created in HighLevel CRM by integrating with Lob's address verification API. It ensures that mailing addresses stored in HighLevel are accurate and deliverable, reducing errors from misspellings or invalid addresses.

**Target Use Case:**  
HighLevel users who want to automatically verify and flag mailing addresses for new contacts added to their CRM, enabling improved mailing accuracy and follow-up actions for invalid addresses.

**Logical Blocks:**  
- **1.1 Input Reception:** Listen for new contact creation events from HighLevel via a webhook.  
- **1.2 Address Preparation:** Extract and structure mailing address data from the webhook payload.  
- **1.3 Address Verification:** Send the address data to Lob’s US address verification API.  
- **1.4 Deliverability Decision:** Evaluate Lob’s response to determine if the address is deliverable.  
- **1.5 HighLevel Update:** Update the contact in HighLevel with tags indicating deliverability status.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives webhook POST requests from HighLevel whenever a new contact is created. This triggers the workflow and passes the contact’s data (including address fields) for processing.

- **Nodes Involved:**  
  - CRM Webhook Trigger

- **Node Details:**  

  - **CRM Webhook Trigger**  
    - *Type:* Webhook Trigger  
    - *Role:* Entry point for the workflow, receiving HighLevel contact data via HTTP POST.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Webhook Path: Fixed unique path (`727deb6f-9d10-4492-92e6-38f3292510b0`)  
    - *Key Expressions:* None, raw JSON payload expected with fields like `address`, `city`, `state`, `zip_code`, `email`, `phone`.  
    - *Input:* External POST request from HighLevel  
    - *Output:* Emits JSON containing contact data to next node  
    - *Edge cases:*  
      - Missing or malformed payload could cause downstream failures.  
      - Webhook timeout or unauthorized calls should be handled externally.  
    - *Version requirements:* n8n webhook node v1.1 or higher recommended.

---

#### 1.2 Address Preparation

- **Overview:**  
Extracts relevant address fields from the webhook data and formats them for Lob’s API requirements.

- **Nodes Involved:**  
  - Set Address Fields

- **Node Details:**  

  - **Set Address Fields**  
    - *Type:* Set Node  
    - *Role:* Maps and prepares address fields into specific variables for the Lob API.  
    - *Configuration:*  
      - Assigns:  
        - `address` = `{{$json.address}}`  
        - `address2` = `""` (empty string, default)  
        - `city` = `{{$json.city}}`  
        - `state` = `{{$json.state}}`  
        - `zip` = `{{$json.zip_code}}`  
      - Includes all other fields from incoming JSON to preserve context.  
    - *Input:* JSON from webhook with full contact data  
    - *Output:* Reformatted JSON with standardized address fields  
    - *Edge cases:*  
      - Missing fields (e.g., empty city or state) could cause Lob API rejection.  
      - Empty `address2` is set by default, assuming secondary address line is optional.  
    - *Version:* n8n Set node v3.3 or above.

---

#### 1.3 Address Verification

- **Overview:**  
Sends the formatted address data to Lob’s US address verification API to check deliverability.

- **Nodes Involved:**  
  - Address Verification

- **Node Details:**  

  - **Address Verification**  
    - *Type:* HTTP Request  
    - *Role:* Calls Lob API to verify the mailing address.  
    - *Configuration:*  
      - HTTP Method: POST  
      - URL: `https://api.lob.com/v1/us_verifications`  
      - Authentication: Basic Auth using Lob API key (configured via credentials)  
      - Body Parameters (JSON):  
        - `primary_line`: `{{$json.address}}`  
        - `secondary_line`: `{{$json.address2}}`  
        - `city`: `{{$json.city}}`  
        - `state`: `{{$json.state}}`  
        - `zip_code`: `{{$json.zip_code}}`  
    - *Input:* JSON with standardized address fields  
    - *Output:* Lob API JSON response including `deliverability` status (e.g., "deliverable", "undeliverable")  
    - *Edge cases:*  
      - API failures such as authentication errors, rate limits, or network timeouts.  
      - Invalid or incomplete address data causing API errors.  
      - Non-US addresses will not be verified correctly (Lob US Verification only).  
    - *Version:* n8n HTTP Request node v4.1 or higher  
    - *Credential Requirements:* Lob API key with Basic Auth

---

#### 1.4 Deliverability Decision

- **Overview:**  
Evaluates the `deliverability` field from Lob’s response and routes the workflow to update HighLevel accordingly.

- **Nodes Involved:**  
  - Switch

- **Node Details:**  

  - **Switch**  
    - *Type:* Conditional Switch  
    - *Role:* Routes execution based on address deliverability status.  
    - *Configuration:*  
      - Input Expression: `{{$json.deliverability}}` (string)  
      - Rules:  
        - If value equals `"deliverable"` → output key "deliverable"  
        - If value not equal `"deliverable"` → output key "NOT deliverable"  
    - *Input:* Lob API JSON response with `deliverability` field  
    - *Output:* Routes to either "deliverable" path or "not deliverable" path  
    - *Edge cases:*  
      - Unexpected `deliverability` values may default to "NOT deliverable" path.  
      - Missing `deliverability` field leads to routing failures, consider fallback handling.

---

#### 1.5 HighLevel Update

- **Overview:**  
Updates the contact in HighLevel CRM by applying tags indicating whether the address was verified as deliverable or not, enabling downstream automation or manual review.

- **Nodes Involved:**  
  - Update HighLevel - Deliverable  
  - Update HighLevel - NOT Deliverable

- **Node Details:**  

  - **Update HighLevel - Deliverable**  
    - *Type:* HighLevel Node  
    - *Role:* Updates contact with tag "Mailing Address Deliverable" for valid addresses.  
    - *Configuration:*  
      - Contact identified by email and phone from webhook data:  
        - Email: `{{$node["CRM Webhook Trigger"].json.email}}`  
        - Phone: `{{$node["CRM Webhook Trigger"].json.phone}}`  
      - Additional Fields: Tags = "Mailing Address Deliverable"  
    - *Input:* Routed from Switch node when address is deliverable  
    - *Output:* None (end node)  
    - *Credentials:* HighLevel API key configured in n8n credentials (OAuth2 or API key)  
    - *Edge cases:*  
      - Contact not found or mismatch in email/phone may cause update failure.  
      - API rate limits or connectivity issues.  

  - **Update HighLevel - NOT Deliverable**  
    - *Type:* HighLevel Node  
    - *Role:* Updates contact with tag "Mailing Address NOT Deliverable" for invalid addresses.  
    - *Configuration:*  
      - Contact identified same as above  
      - Additional Fields: Tags = "Mailing Address NOT Deliverable"  
    - *Input:* Routed from Switch node when address is not deliverable  
    - *Output:* None (end node)  
    - *Credentials:* Same as above  
    - *Edge cases:* Same as above  

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                                | Input Node(s)        | Output Node(s)                      | Sticky Note                                                                                                      |
|------------------------------|----------------------|------------------------------------------------|----------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------|
| CRM Webhook Trigger           | Webhook Trigger      | Receives new contact data from HighLevel       | -                    | Set Address Fields                | Receive a webhook from your CRM with the contact address fields                                                 |
| Set Address Fields            | Set                  | Prepare and format address fields for Lob API  | CRM Webhook Trigger  | Address Verification             |                                                                                                                  |
| Address Verification          | HTTP Request         | Verify address deliverability using Lob API    | Set Address Fields    | Switch                          | 1. Create an Account a LOB.com 2. Create API Key (https://help.lob.com/account-management/api-keys) 3. Update Node with your Credentials (Basic Auth) |
| Switch                       | Switch               | Routes workflow based on deliverability status | Address Verification | Update HighLevel - Deliverable, Update HighLevel - NOT Deliverable |                                                                                                                  |
| Update HighLevel - Deliverable| HighLevel            | Tag contact as address deliverable              | Switch (deliverable)  | -                               | Update HighLevel to indicate if the address is deliverable. You could: Add Tag, Start Automation, Update a Field; For Deliverable Addresses - I apply a tag that the address was verified. |
| Update HighLevel - NOT Deliverable | HighLevel        | Tag contact as address NOT deliverable          | Switch (NOT deliverable) | -                               | Update HighLevel to indicate if the address is deliverable. You could: Add Tag, Start Automation, Update a Field; For Non Deliverable Addresses - I apply a tag, which triggers an automation for my team to manually verify the address. You could also trigger an automation to reach out to the contact to verify their address. |
| Sticky Note                  | Sticky Note           | Purpose overview and video link                  | -                    | -                               | ## Purpose To verify the mailing address for new contacts in HighLevel. Whenever I add a new contact to HighLevel, I run this automation to ensure I have a valid mailing address. It also helps me check for misspellings if the contact address was manually entered. Quick Video Overview: https://www.loom.com/share/8995ca0b41ce473ebbad9c1973109c0f |
| Sticky Note1                 | Sticky Note           | Guidance on updating HighLevel based on deliverability | -                    | -                               | Update HighLevel to indicate if the address is deliverable. You could: Add Tag, Start Automation, Update a Field For Deliverable Addresses - I apply a tag that the address was verified. For Non Deliverable Addresses - I apply a tag, which triggers an automation for my team to manually verify the address. You could also trigger an automation to reach out to the contact to verify their address. |
| Sticky Note2                 | Sticky Note           | Explains webhook reception                      | -                    | -                               | Receive a webhook from your CRM with the contact address fields                                                  |
| Sticky Note3                 | Sticky Note           | Lob setup instructions                          | -                    | -                               | 1. Create an Account a LOB.com 2. Create API Key (https://help.lob.com/account-management/api-keys) 3. Update Node with your Credentials (Basic Auth) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Type: Webhook Trigger  
   - HTTP Method: POST  
   - Path: `727deb6f-9d10-4492-92e6-38f3292510b0` (or any unique string)  
   - Purpose: Receive new contact data from HighLevel CRM webhook.

2. **Add Set Node to Prepare Address Fields:**  
   - Name: Set Address Fields  
   - Assign fields as follows:  
     - `address`: Expression: `{{$json.address}}`  
     - `address2`: Set as empty string `""`  
     - `city`: Expression: `{{$json.city}}`  
     - `state`: Expression: `{{$json.state}}`  
     - `zip`: Expression: `{{$json.zip_code}}`  
   - Enable "Include Other Fields" to keep all original fields for later use.

3. **Add HTTP Request Node for Lob Address Verification:**  
   - Name: Address Verification  
   - HTTP Method: POST  
   - URL: `https://api.lob.com/v1/us_verifications`  
   - Authentication: Basic Auth with Lob API key (configure credentials in n8n)  
   - Body Parameters (as JSON):  
     - `primary_line`: `{{$json.address}}`  
     - `secondary_line`: `{{$json.address2}}`  
     - `city`: `{{$json.city}}`  
     - `state`: `{{$json.state}}`  
     - `zip_code`: `{{$json.zip}}`  
   - Set to send body as JSON.

4. **Add Switch Node to Route by Deliverability:**  
   - Name: Switch  
   - Input value: `{{$json.deliverability}}`  
   - Rules:  
     - If equals `"deliverable"` → output 1  
     - Else → output 2

5. **Add HighLevel Node to Tag Deliverable Addresses:**  
   - Name: Update HighLevel - Deliverable  
   - Configure credentials with HighLevel API key  
   - Operation: Update Contact (or equivalent)  
   - Identify contact by:  
     - Email: `{{$node["CRM Webhook Trigger"].json.email}}`  
     - Phone: `{{$node["CRM Webhook Trigger"].json.phone}}`  
   - Additional Fields: Add Tag "Mailing Address Deliverable"

6. **Add HighLevel Node to Tag Non-Deliverable Addresses:**  
   - Name: Update HighLevel - NOT Deliverable  
   - Same contact identification as above  
   - Additional Fields: Add Tag "Mailing Address NOT Deliverable"

7. **Connect Nodes in Order:**  
   - CRM Webhook Trigger → Set Address Fields → Address Verification → Switch  
   - Switch output "deliverable" → Update HighLevel - Deliverable  
   - Switch output "NOT deliverable" → Update HighLevel - NOT Deliverable

8. **Set up Credentials:**  
   - Lob: Create an account at https://www.lob.com, generate API key, configure Basic Auth credentials in n8n.  
   - HighLevel: Obtain API key or OAuth credentials, configure in n8n.

9. **Test the Workflow:**  
   - Trigger a webhook POST from HighLevel with a new contact including valid address data.  
   - Verify that the workflow flags the contact correctly and tags them in HighLevel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| Quick Video Overview of the workflow and setup: https://www.loom.com/share/8995ca0b41ce473ebbad9c1973109c0f                                               | Introductory video overview                   |
| Setup video explaining step-by-step how to configure the workflow: https://www.youtube.com/watch?v=T7Baopubc-0                                             | Workflow setup instructions                    |
| Lob API key creation and account management: https://help.lob.com/account-management/api-keys                                                               | Lob API key setup documentation                |
| Workflow takes approximately 10-30 minutes to fully configure and test                                                                                     | Estimated setup time                           |
| Recommended to customize trigger and actions based on specific HighLevel workflow automation needs                                                         | Customization advice                           |
| For undeliverable addresses, consider triggering automations to notify contacts or teams for manual verification                                          | Suggested automation enhancement               |
| Lob US Address Verification API only supports US addresses and may not validate international addresses accurately                                        | API limitation note                            |