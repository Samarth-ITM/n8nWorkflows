Send contract renewal reminders from HubSpot via Gmail and Slack alerts

https://n8nworkflows.xyz/workflows/send-contract-renewal-reminders-from-hubspot-via-gmail-and-slack-alerts-14155


# Send contract renewal reminders from HubSpot via Gmail and Slack alerts

# 1. Workflow Overview

This workflow automates contract renewal reminders based on HubSpot deal expiry dates. It runs on a schedule, retrieves deals from HubSpot, calculates how many days remain until each contract expires, keeps only deals expiring in exactly 30, 60, or 90 days, then fetches the associated contact, sends a tailored Gmail reminder, alerts the account manager in Slack, and creates a ClickUp follow-up task.

Typical use cases:
- Customer success or sales operations teams managing contract renewals
- Internal retention workflows with proactive outreach
- Renewal reminder systems tied to CRM deal data

## 1.1 Scheduled deal retrieval and expiry filtering
The workflow starts from a scheduled trigger, retrieves all HubSpot deals with contract expiry metadata, and filters the dataset down to deals expiring in 30, 60, or 90 days.

## 1.2 Per-deal processing and contact lookup
Each filtered deal is processed one at a time through a loop. For every deal, the workflow fetches the contact associated with the deal and then retrieves contact details from HubSpot.

## 1.3 Reminder routing and email delivery
A Switch node checks the computed `daysLeft` value and routes each deal to one of three Gmail nodes, each containing different HTML messaging for 30-day, 60-day, and 90-day reminders.

## 1.4 Internal alerting and follow-up task creation
After an email is sent, the branches are merged and passed to Slack for an internal alert. A ClickUp task is then created for follow-up, and the loop continues with the next deal.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled deal retrieval and expiry filtering

**Overview:**  
This block initiates the workflow automatically and prepares the list of relevant deals. It extracts all deals from HubSpot, computes the number of days left until contract expiry, and keeps only deals matching the reminder thresholds.

**Nodes Involved:**  
- Schedule Trigger
- Get all deals
- Filter Deals

### Node: Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Entry point that runs the workflow on a recurring schedule.
- **Configuration choices:**  
  Uses an interval rule with default interval settings from the template. The JSON does not specify a custom cadence, so the actual schedule should be reviewed and adjusted after import.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **Get all deals**.
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Misconfigured interval may cause the workflow to run too often or not often enough
  - Timezone differences may affect expected execution time
- **Sub-workflow reference:**  
  None.

### Node: Get all deals
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Retrieves HubSpot deals in bulk.
- **Configuration choices:**  
  - Resource: `deal`
  - Operation: `getAll`
  - Authentication: `appToken`
  - Requests the properties `contract_expiry_date` and `dealname`
  - Does not include associations
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Input from **Schedule Trigger**. Output to **Filter Deals**.
- **Version-specific requirements:**  
  Type version `2.2`. Requires HubSpot credentials configured as app token in n8n.
- **Edge cases or potential failure types:**  
  - HubSpot authentication failure
  - Missing deal properties if `contract_expiry_date` is absent on some records
  - Pagination/load concerns if there are many deals
  - Rate limiting from HubSpot
- **Sub-workflow reference:**  
  None.

### Node: Filter Deals
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that calculates `daysLeft` from the HubSpot contract expiry timestamp and filters matching deals.
- **Configuration choices:**  
  The code:
  - Sets `today` to local midnight
  - Reads `item.json.properties?.contract_expiry_date?.value`
  - Converts timestamp to a Date
  - Normalizes expiry date to midnight
  - Computes day difference
  - Returns simplified objects:
    - `dealId`
    - `dealName`
    - `expiryDate`
    - `daysLeft`
  - Filters for `daysLeft` in `[30, 60, 90]`
- **Key expressions or variables used:**  
  - `item.json.properties?.contract_expiry_date?.value`
  - `item.json.dealId`
  - `item.json.properties?.dealname?.value`
- **Input and output connections:**  
  Input from **Get all deals**. Output to **Loop Over Deals**.
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Invalid or missing `contract_expiry_date`
  - Timestamp format assumptions: code expects a numeric timestamp
  - Timezone drift may shift the calculated day count by 1
  - `Math.round()` can create off-by-one behavior depending on timestamp precision and timezone normalization
- **Sub-workflow reference:**  
  None.

---

## 2.2 Per-deal processing and contact lookup

**Overview:**  
This block iterates through each filtered deal, fetches the deal-to-contact association through HubSpot’s API, and then retrieves the contact record itself for email delivery and personalization.

**Nodes Involved:**  
- Loop Over Deals
- Fetch Associated Contact With Deal
- Get Contact Details

### Node: Loop Over Deals
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the filtered deals one by one.
- **Configuration choices:**  
  Uses default batching behavior. In this design, the loop continues after the ClickUp task is created.
- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `$('Loop Over Deals').item.json.dealId`
  - `$('Loop Over Deals').item.json.dealName`
  - `$('Loop Over Deals').item.json.daysLeft`
- **Input and output connections:**  
  Input from **Filter Deals**.  
  Loop branch output goes to **Fetch Associated Contact With Deal**.  
  Completion path is not used.  
  Receives continuation input from **Create Follow-up Task** back into the loop.
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If no deals pass the filter, nothing is processed
  - Loop behavior depends on correct back-connection from the final node
  - Very large numbers of deals may increase execution duration
- **Sub-workflow reference:**  
  None.

### Node: Fetch Associated Contact With Deal
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Makes a direct HubSpot API call to retrieve contacts associated with the current deal.
- **Configuration choices:**  
  - URL: `https://api.hubapi.com/crm/v3/objects/deals/{{$json["dealId"]}}/associations/contacts`
  - Sends custom headers
  - Authorization header value is hardcoded as placeholder: `YOUR_HUBSPOT_API_KEY`
- **Key expressions or variables used:**  
  - `{{$json["dealId"]}}`
- **Input and output connections:**  
  Input from **Loop Over Deals**. Output to **Get Contact Details**.
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Placeholder auth header must be replaced
  - If using a private app token, HubSpot typically expects `Authorization: Bearer <token>`
  - Deal may have no associated contacts
  - API may return multiple contacts; this node does not choose explicitly beyond downstream first result usage
  - HTTP 401/403/404/429 errors
- **Sub-workflow reference:**  
  None.

### Node: Get Contact Details
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Retrieves a single HubSpot contact based on the first associated contact ID.
- **Configuration choices:**  
  - Operation: `get`
  - Authentication: `appToken`
  - Contact ID taken from `{{$json.results[0].id}}`
  - Additional fields request property `email`
- **Key expressions or variables used:**  
  - `={{ $json.results[0].id }}`
- **Input and output connections:**  
  Input from **Fetch Associated Contact With Deal**. Output to **Switch**.
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If `results` is empty, the expression fails
  - The node only requests `email`, but downstream expressions also reference `hs_full_name_or_email` and `identity-profiles`
  - Contact may not have an email
  - Authentication/rate-limit errors in HubSpot
- **Sub-workflow reference:**  
  None.

---

## 2.3 Reminder routing and email delivery

**Overview:**  
This block determines whether the deal is 30, 60, or 90 days from expiry and sends the corresponding Gmail reminder. It then merges the branches to continue the workflow uniformly.

**Nodes Involved:**  
- Switch
- 30 day mail
- 60 day mail
- 90 day mail
- Merge

### Node: Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes items to one of three outputs based on `daysLeft`.
- **Configuration choices:**  
  Three conditions:
  1. `daysLeft == 30`
  2. `daysLeft == 60`
  3. `daysLeft == 90`
  
  The configuration mixes numeric and string comparison styles:
  - 30 uses numeric equals
  - 60 and 90 use string equals
  - `looseTypeValidation` is enabled, so this still works in many cases
- **Key expressions or variables used:**  
  - `={{ $('Loop Over Deals').item.json.daysLeft }}`
- **Input and output connections:**  
  Input from **Get Contact Details**.  
  Output 0 to **30 day mail**  
  Output 1 to **60 day mail**  
  Output 2 to **90 day mail**
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Mixed type comparisons may be confusing during maintenance
  - If `daysLeft` is undefined, no branch matches
  - If the expression context changes, branching may break
- **Sub-workflow reference:**  
  None.

### Node: 30 day mail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the 30-day renewal reminder email using Gmail.
- **Configuration choices:**  
  - Subject: `Action Required: Contract Expiring in 30 Days`
  - HTML body uses urgency-focused language
  - Recipient is pulled from `identity-profiles[0].identities[0].value`
- **Key expressions or variables used:**  
  - `={{ $json["identity-profiles"][0].identities[0].value }}`
  - `{{ $json.properties.hs_full_name_or_email.value }}`
  - `{{ $('Loop Over Deals').item.json.dealName }}`
  - `{{ $('Loop Over Deals').item.json.daysLeft }}`
- **Input and output connections:**  
  Input from **Switch** output 0. Output to **Merge** input 0.
- **Version-specific requirements:**  
  Type version `2.2`. Requires Gmail credentials in n8n.
- **Edge cases or potential failure types:**  
  - `identity-profiles` may not exist in contact output
  - `hs_full_name_or_email` may not have been requested by the HubSpot node
  - Gmail auth or sending quota issues
  - Invalid recipient email
- **Sub-workflow reference:**  
  None.

### Node: 60 day mail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the 60-day renewal reminder email using Gmail.
- **Configuration choices:**  
  - Subject: `Action Required: Contract Expiring in 60 Days`
  - HTML body uses a friendly reminder tone
  - Recipient uses the same `identity-profiles` path as the 30-day branch
- **Key expressions or variables used:**  
  - `={{ $json["identity-profiles"][0].identities[0].value }}`
  - `{{ $json.properties.hs_full_name_or_email.value }}`
  - `{{ $('Loop Over Deals').item.json.dealName }}`
  - `{{ $('Loop Over Deals').item.json.daysLeft }}`
- **Input and output connections:**  
  Input from **Switch** output 1. Output to **Merge** input 1.
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  Same as **30 day mail**.
- **Sub-workflow reference:**  
  None.

### Node: 90 day mail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the 90-day reminder email using Gmail.
- **Configuration choices:**  
  - Subject: `Action Required: Contract Expiring in 90 Days`
  - HTML body uses a soft early notice tone
  - Uses `$('Loop Over Deals').first()` instead of `.item` for deal references
- **Key expressions or variables used:**  
  - `={{ $json["identity-profiles"][0].identities[0].value }}`
  - `{{ $json.properties.hs_full_name_or_email.value }}`
  - `{{ $('Loop Over Deals').first().json.dealName }}`
  - `{{ $('Loop Over Deals').first().json.daysLeft }}`
- **Input and output connections:**  
  Input from **Switch** output 2. Output to **Merge** input 2.
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - `first()` may pull the first loop item, not the current one, causing wrong content in multi-item runs
  - Same contact-property risks as other Gmail nodes
  - Gmail auth and recipient issues
- **Sub-workflow reference:**  
  None.

### Node: Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Consolidates the three email branches into one flow before internal alerting.
- **Configuration choices:**  
  Configured for `numberInputs: 3`.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  Inputs from **30 day mail**, **60 day mail**, and **90 day mail**. Output to **Nofity Account Manager**.
- **Version-specific requirements:**  
  Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Merge behavior depends on n8n’s selected merge mode defaults for this version
  - Branches without data simply do not contribute items
- **Sub-workflow reference:**  
  None.

---

## 2.4 Internal alerting and follow-up task creation

**Overview:**  
This block informs the account manager internally through Slack and creates a ClickUp task so the renewal can be followed up operationally. It then returns control to the loop for the next deal.

**Nodes Involved:**  
- Nofity Account Manager
- Create Follow-up Task

### Node: Nofity Account Manager
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a Slack message to notify the internal team or account manager about the expiring contract.
- **Configuration choices:**  
  - Authentication: `oAuth2`
  - Select mode: `user`
  - User field is empty in the JSON, so the actual Slack recipient/channel is not yet configured
  - Message body includes contact, deal name, and days left
- **Key expressions or variables used:**  
  - `{{ $('Get Contact Details').item.json.properties.hs_full_name_or_email.value }}`
  - `{{ $('Loop Over Deals').item.json.dealName }}`
  - `{{ $('Loop Over Deals').item.json.daysLeft }}`
- **Input and output connections:**  
  Input from **Merge**. Output to **Create Follow-up Task**.
- **Version-specific requirements:**  
  Type version `2.4`. Requires Slack OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Node name contains a typo: “Nofity”
  - Empty Slack user selection may cause failure
  - `hs_full_name_or_email` may not be present
  - Slack permissions may be insufficient
  - OAuth token expiry or workspace restrictions
- **Sub-workflow reference:**  
  None.

### Node: Create Follow-up Task
- **Type and technical role:** `n8n-nodes-base.clickUp`  
  Creates a task in ClickUp to track contract renewal follow-up.
- **Configuration choices:**  
  - Team, space, folder, list are all placeholders:
    - `YOUR_CLICKUP_TEAM_ID`
    - `YOUR_CLICKUP_SPACE_ID`
    - `YOUR_CLICKUP_FOLDER_ID`
    - `YOUR_CLICKUP_LIST_ID`
  - Task name: `Contract Renewal Follow-up: {{ dealName }}`
- **Key expressions or variables used:**  
  - `=Contract Renewal Follow-up: {{ $('Loop Over Deals').item.json.dealName }} `
- **Input and output connections:**  
  Input from **Nofity Account Manager**. Output loops back to **Loop Over Deals**.
- **Version-specific requirements:**  
  Type version `1`. Requires ClickUp credentials.
- **Edge cases or potential failure types:**  
  - Placeholder IDs must be replaced
  - ClickUp auth failures
  - Invalid hierarchy combination: team/space/folder/list mismatch
  - Duplicate task creation if the workflow is rerun for the same deal without deduplication logic
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Get all deals | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 1: Schedule & Filter Contracts\nRuns on a schedule, fetches deals from HubSpot, and filters contracts expiring in 30, 60, or 90 days. |
| Get all deals | HubSpot | Fetches all deals and selected deal properties from HubSpot | Schedule Trigger | Filter Deals | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 1: Schedule & Filter Contracts\nRuns on a schedule, fetches deals from HubSpot, and filters contracts expiring in 30, 60, or 90 days. |
| Filter Deals | Code | Calculates days until expiry and keeps only 30/60/90 day deals | Get all deals | Loop Over Deals | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 1: Schedule & Filter Contracts\nRuns on a schedule, fetches deals from HubSpot, and filters contracts expiring in 30, 60, or 90 days. |
| Loop Over Deals | Split In Batches | Iterates through filtered deals one by one | Filter Deals, Create Follow-up Task | Fetch Associated Contact With Deal | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 2: Process Deals & Fetch Contacts\nLoops through deals, fetches associated contacts, and retrieves email details for communication. |
| Fetch Associated Contact With Deal | HTTP Request | Gets deal-to-contact associations from HubSpot API | Loop Over Deals | Get Contact Details | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 2: Process Deals & Fetch Contacts\nLoops through deals, fetches associated contacts, and retrieves email details for communication. |
| Get Contact Details | HubSpot | Retrieves the associated contact record | Fetch Associated Contact With Deal | Switch | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 2: Process Deals & Fetch Contacts\nLoops through deals, fetches associated contacts, and retrieves email details for communication. |
| Switch | Switch | Routes each deal to the correct reminder branch | Get Contact Details | 30 day mail, 60 day mail, 90 day mail | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 3: Email Routing & Sending\nUses a Switch node to route deals (30/60/90 days) and sends personalized emails via Gmail. |
| 30 day mail | Gmail | Sends the 30-day contract reminder email | Switch | Merge | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 3: Email Routing & Sending\nUses a Switch node to route deals (30/60/90 days) and sends personalized emails via Gmail. |
| 60 day mail | Gmail | Sends the 60-day contract reminder email | Switch | Merge | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 3: Email Routing & Sending\nUses a Switch node to route deals (30/60/90 days) and sends personalized emails via Gmail. |
| 90 day mail | Gmail | Sends the 90-day contract reminder email | Switch | Merge | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 3: Email Routing & Sending\nUses a Switch node to route deals (30/60/90 days) and sends personalized emails via Gmail. |
| Merge | Merge | Rejoins email branches into one path | 30 day mail, 60 day mail, 90 day mail | Nofity Account Manager | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 3: Email Routing & Sending\nUses a Switch node to route deals (30/60/90 days) and sends personalized emails via Gmail. |
| Nofity Account Manager | Slack | Sends internal Slack alert after email delivery | Merge | Create Follow-up Task | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 4: Alerts & Task Creation\nSends Slack alerts to account managers and creates ClickUp tasks for follow-up actions. |
| Create Follow-up Task | ClickUp | Creates a ClickUp renewal follow-up task | Nofity Account Manager | Loop Over Deals | # Contract Expiry & Renewal Alert System\nThis workflow automates contract renewal management by tracking upcoming expirations and triggering timely actions. It ensures that no contract is missed by proactively notifying clients and internal teams, improving retention and operational efficiency.\nHow it works: Runs on a schedule to fetch all deals from HubSpot; Filters contracts expiring in 30, 60, or 90 days; Loops through each deal and fetches associated contact details; Routes deals using a Switch node based on days left; Sends automated email reminders via Gmail; Notifies account managers in Slack; Creates follow-up tasks in ClickUp for tracking.\nSetup steps: Connect your HubSpot account (Deals + Contacts access required); Configure Gmail credentials for sending emails; Add Slack credentials for internal notifications; Connect ClickUp API for task creation; Replace any placeholder API keys (e.g., HubSpot HTTP node); Adjust schedule trigger timing based on your needs.\n## Step 4: Alerts & Task Creation\nSends Slack alerts to account managers and creates ClickUp tasks for follow-up actions. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name it: `Schedule Trigger`
   - Configure the interval based on your preferred cadence, for example daily at a specific hour.
   - This is the workflow entry point.

3. **Add a HubSpot node to fetch deals**
   - Node type: **HubSpot**
   - Name it: `Get all deals`
   - Resource: `Deal`
   - Operation: `Get Many` / `Get All`
   - Authentication: **App Token**
   - Select properties:
     - `contract_expiry_date`
     - `dealname`
   - Leave associations disabled.
   - Connect `Schedule Trigger -> Get all deals`.

4. **Configure HubSpot credentials**
   - Create HubSpot credentials in n8n using a valid private app token or supported HubSpot app token method.
   - Ensure permissions include:
     - Deals read
     - Contacts read

5. **Add a Code node to compute reminder eligibility**
   - Node type: **Code**
   - Name it: `Filter Deals`
   - Paste logic equivalent to:
     - normalize today to midnight
     - read the HubSpot contract expiry timestamp
     - convert timestamp to date
     - calculate days remaining
     - return only records where days remaining is exactly 30, 60, or 90
   - Output fields should be:
     - `dealId`
     - `dealName`
     - `expiryDate`
     - `daysLeft`
   - Connect `Get all deals -> Filter Deals`.

6. **Add a Split In Batches node**
   - Node type: **Split In Batches**
   - Name it: `Loop Over Deals`
   - Use default batch size unless you want explicit control; one item per iteration is typical here.
   - Connect `Filter Deals -> Loop Over Deals`.

7. **Add an HTTP Request node to fetch associated contacts**
   - Node type: **HTTP Request**
   - Name it: `Fetch Associated Contact With Deal`
   - Method: `GET`
   - URL:
     `https://api.hubapi.com/crm/v3/objects/deals/{{$json["dealId"]}}/associations/contacts`
   - Enable headers.
   - Add an Authorization header.
   - Recommended format:
     - Header name: `Authorization`
     - Header value: `Bearer YOUR_HUBSPOT_PRIVATE_APP_TOKEN`
   - Connect `Loop Over Deals -> Fetch Associated Contact With Deal`.

8. **Optional but recommended improvement**
   - Instead of hardcoding the HubSpot token in the HTTP Request node, use:
     - a proper credential-supported node if available
     - or an expression/environment variable
   - This reduces secret exposure.

9. **Add a HubSpot node to retrieve contact details**
   - Node type: **HubSpot**
   - Name it: `Get Contact Details`
   - Resource: `Contact`
   - Operation: `Get`
   - Authentication: same HubSpot credential
   - Contact ID expression:
     `{{ $json.results[0].id }}`
   - Request additional properties. At minimum, configure:
     - `email`
   - To make the downstream expressions safe, also add:
     - `hs_full_name_or_email`
   - Connect `Fetch Associated Contact With Deal -> Get Contact Details`.

10. **Add a Switch node**
    - Node type: **Switch**
    - Name it: `Switch`
    - Create 3 outputs.
    - Configure conditions using the `daysLeft` value from the loop item:
      - Output 1: `{{ $('Loop Over Deals').item.json.daysLeft }}` equals `30`
      - Output 2: same expression equals `60`
      - Output 3: same expression equals `90`
    - Prefer using numeric comparisons consistently for all three branches.
    - Connect `Get Contact Details -> Switch`.

11. **Add the first Gmail node**
    - Node type: **Gmail**
    - Name it: `30 day mail`
    - Operation: Send email
    - Subject: `Action Required: Contract Expiring in 30 Days`
    - Send To:
      - Prefer `{{ $json.properties.email }}` if available
      - The original workflow uses `{{ $json["identity-profiles"][0].identities[0].value }}`, but this is fragile unless that field is known to exist
    - Message:
      - Use HTML mode
      - Insert contact name/email and deal details with expressions referencing:
        - `{{ $json.properties.hs_full_name_or_email.value }}`
        - `{{ $('Loop Over Deals').item.json.dealName }}`
        - `{{ $('Loop Over Deals').item.json.daysLeft }}`
    - Connect `Switch output 1 -> 30 day mail`.

12. **Add the second Gmail node**
    - Node type: **Gmail**
    - Name it: `60 day mail`
    - Subject: `Action Required: Contract Expiring in 60 Days`
    - Use the same recipient strategy and similar HTML body with a less urgent tone.
    - Connect `Switch output 2 -> 60 day mail`.

13. **Add the third Gmail node**
    - Node type: **Gmail**
    - Name it: `90 day mail`
    - Subject: `Action Required: Contract Expiring in 90 Days`
    - Use the same recipient strategy and a gentle early-warning message.
    - Use `{{ $('Loop Over Deals').item.json.dealName }}` and `{{ $('Loop Over Deals').item.json.daysLeft }}` rather than `.first()` to avoid incorrect values.
    - Connect `Switch output 3 -> 90 day mail`.

14. **Configure Gmail credentials**
    - Create Gmail credentials in n8n.
    - Use OAuth2 authorization.
    - Ensure the connected mailbox is allowed to send the reminders.
    - Watch for sending limits and domain policy restrictions.

15. **Add a Merge node**
    - Node type: **Merge**
    - Name it: `Merge`
    - Set number of inputs to `3`
    - Connect:
      - `30 day mail -> Merge input 1`
      - `60 day mail -> Merge input 2`
      - `90 day mail -> Merge input 3`

16. **Add a Slack node for internal notification**
    - Node type: **Slack**
    - Name it: `Nofity Account Manager` or preferably rename to `Notify Account Manager`
    - Authentication: OAuth2
    - Choose the target carefully:
      - direct message to a user
      - or a channel if your process is team-based
    - Message content should include:
      - contact name/email
      - deal name
      - days left
    - Example expressions:
      - `{{ $('Get Contact Details').item.json.properties.hs_full_name_or_email.value }}`
      - `{{ $('Loop Over Deals').item.json.dealName }}`
      - `{{ $('Loop Over Deals').item.json.daysLeft }}`
    - Connect `Merge -> Notify Account Manager`.

17. **Configure Slack credentials**
    - Create Slack OAuth2 credentials in n8n.
    - Ensure the Slack app has permission to post to the chosen user/channel/workspace target.

18. **Add a ClickUp node**
    - Node type: **ClickUp**
    - Name it: `Create Follow-up Task`
    - Operation: Create task
    - Fill in:
      - Team ID
      - Space ID
      - Folder ID
      - List ID
    - Task name expression:
      `Contract Renewal Follow-up: {{ $('Loop Over Deals').item.json.dealName }}`
    - Optionally add due date, assignee, priority, or description fields.
    - Connect `Notify Account Manager -> Create Follow-up Task`.

19. **Configure ClickUp credentials**
    - Create ClickUp credentials in n8n.
    - Confirm the selected team/space/folder/list hierarchy is valid.
    - Replace all placeholders with real IDs.

20. **Close the loop**
    - Connect `Create Follow-up Task -> Loop Over Deals`
    - This allows the Split In Batches node to process the next item after the current one is completed.

21. **Test the workflow with sample data**
    - Verify:
      - deals are returned
      - `contract_expiry_date` is in the expected timestamp format
      - associations return at least one contact
      - contact email property is populated
      - the correct switch branch is chosen
      - Gmail sends properly
      - Slack receives the message
      - ClickUp task is created

22. **Harden the workflow before production**
    - Add handling for deals with no associated contacts
    - Add handling for contacts with no email
    - Standardize Switch comparisons as numbers
    - Replace `identity-profiles` email lookup with `properties.email` unless you explicitly need identity profiles
    - Replace `first()` in the 90-day email with `item`
    - Consider deduplication so repeated daily runs do not create duplicate ClickUp tasks or duplicate reminders for the same deal on the same day

### Sub-workflow setup
This workflow does **not** include any sub-workflow or Execute Workflow node. There are no child workflows to configure.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Contract Expiry & Renewal Alert System: automates contract renewal management by tracking upcoming expirations and triggering timely actions. | Overall workflow purpose |
| How it works: runs on a schedule to fetch all deals from HubSpot; filters contracts expiring in 30, 60, or 90 days; loops through each deal and fetches associated contact details; routes deals using a Switch node based on days left; sends automated email reminders via Gmail; notifies account managers in Slack; creates follow-up tasks in ClickUp for tracking. | Overall workflow logic |
| Setup steps noted in the workflow: connect HubSpot account with Deals and Contacts access; configure Gmail credentials; add Slack credentials; connect ClickUp API; replace placeholder API keys such as the HubSpot HTTP node authorization value; adjust schedule timing as needed. | Deployment and configuration guidance |
| Step 1: Schedule & Filter Contracts | Internal workflow annotation |
| Step 2: Process Deals & Fetch Contacts | Internal workflow annotation |
| Step 3: Email Routing & Sending | Internal workflow annotation |
| Step 4: Alerts & Task Creation | Internal workflow annotation |

## Additional implementation notes
- The current workflow mixes credential-based HubSpot access with a hardcoded HTTP Authorization placeholder. Standardizing authentication is recommended.
- The Gmail nodes rely on contact fields that are not safely guaranteed by the current `Get Contact Details` configuration. Using `properties.email` and explicitly fetching `hs_full_name_or_email` is safer.
- The 90-day email branch uses `first()` instead of `item`, which may cause incorrect personalization in multi-item runs.
- The Slack node target is incomplete in the provided JSON and must be configured before production use.
- The ClickUp node contains placeholder hierarchy IDs and cannot run successfully until they are replaced.