Sync Event Attendees from Eventbrite to Pipedrive Automatically

https://n8nworkflows.xyz/workflows/sync-event-attendees-from-eventbrite-to-pipedrive-automatically-6712


# Sync Event Attendees from Eventbrite to Pipedrive Automatically

### 1. Workflow Overview

This workflow automatically synchronizes event attendees from Eventbrite to Pipedrive as new leads. Its primary use case is for organizations hosting events on Eventbrite who want to keep their Pipedrive CRM updated with fresh attendee contact details without duplicates. The workflow consists of the following logical blocks:

- **1.1 Trigger and Initiation:** Scheduled or manual start of the synchronization process.
- **1.2 Eventbrite Data Extraction:** Retrieve all events and their attendees from Eventbrite using the API.
- **1.3 Pipedrive Leads Extraction:** Fetch all existing leads (persons) currently in Pipedrive.
- **1.4 Data Merging and Filtering:** Compare Eventbrite attendees against existing Pipedrive leads to isolate new leads.
- **1.5 Lead Creation in Pipedrive:** Add only the filtered new attendees as leads in Pipedrive.

The workflow is designed to run on a schedule (default every 10 minutes) or manually triggered for testing or manual syncs.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initiation

- **Overview:**  
  This block starts the workflow either on a scheduled interval or manually on demand.

- **Nodes Involved:**  
  - Schedule Daily  
  - Test or Manually run workflow

- **Node Details:**

  - **Schedule Daily**  
    - *Type & Role:* Trigger node that initiates workflow execution on a recurring schedule.  
    - *Configuration:* Set to trigger every 10 minutes by default; configurable to other intervals (hours, days).  
    - *Input/Output:* No input; outputs to "Extract Eventbrite Signups" and "Extract current leads in pipedrive" nodes.  
    - *Potential Failures:* Misconfigured schedule or n8n instance downtime may delay runs.

  - **Test or Manually run workflow**  
    - *Type & Role:* Manual trigger node allowing ad-hoc workflow runs for testing.  
    - *Configuration:* No parameters.  
    - *Input/Output:* No input; outputs connected similarly to Schedule Daily node.  
    - *Potential Failures:* None inherent; manual runs depend on user action.

---

#### 1.2 Eventbrite Data Extraction

- **Overview:**  
  Fetches all events (live and ended) for the configured Eventbrite organization and retrieves all attendees per event via the Eventbrite API. Extracted attendee data is structured for downstream processing.

- **Nodes Involved:**  
  - Extract Eventbrite Signups

- **Node Details:**

  - **Extract Eventbrite Signups**  
    - *Type & Role:* Code node executing custom JavaScript to call Eventbrite REST APIs, paginate through events and attendees, and collect attendee info into a JSON array.  
    - *Configuration Highlights:*  
      - Hardcoded Eventbrite OAuth token and organization ID must be replaced with real values.  
      - Uses asynchronous requests to fetch paginated events and their attendees.  
      - Extracts selected attendee fields including event ID, name, email, phone numbers, ticket class, status, order ID, and creation date.  
      - Custom answers from Eventbrite attendee questions are parsed but not currently integrated into downstream data.  
    - *Key Expressions/Variables:*  
      - `token` (OAuth token), `orgId` (organization ID), `allAttendees` (accumulated attendee data array).  
      - Pagination handled with `continuation` tokens for both event and attendee lists.  
    - *Input/Output:* No input; outputs a list of attendee JSON objects.  
    - *Failures & Edge Cases:*  
      - Invalid or expired token results in auth errors.  
      - API rate limits or network issues may cause timeouts or incomplete data.  
      - If Eventbrite API structure changes, parsing may fail.  
      - Large event or attendee volumes may increase execution time.  
    - *Version Requirements:* n8n version supporting asynchronous Code node execution (v0.140+ recommended).  
    - *Sub-workflow:* None.

---

#### 1.3 Pipedrive Leads Extraction

- **Overview:**  
  Retrieves all current leads ("persons") from Pipedrive to enable deduplication by comparing existing contacts with newly fetched Eventbrite attendees.

- **Nodes Involved:**  
  - Extract current leads in pipedrive

- **Node Details:**

  - **Extract current leads in pipedrive**  
    - *Type & Role:* Pipedrive node performing a "getAll" operation on the "person" resource via Pipedrive API.  
    - *Configuration Highlights:*  
      - Returns all existing person records, no filters applied.  
      - Uses configured Pipedrive API credentials.  
    - *Key Expressions:* None dynamic; direct API call.  
    - *Input/Output:* No input; outputs full list of Pipedrive persons.  
    - *Failures & Edge Cases:*  
      - Authentication failures if credentials expire or are revoked.  
      - Large datasets may cause slow responses or timeouts.  
    - *Version Requirements:* Compatible with n8n Pipedrive node version 1+.  
    - *Sub-workflow:* None.

---

#### 1.4 Data Merging and Filtering

- **Overview:**  
  Compares Eventbrite attendee emails against Pipedrive lead primary emails to isolate attendees not yet in Pipedrive, preventing duplicate lead creation.

- **Nodes Involved:**  
  - Merge to keep only new leads

- **Node Details:**

  - **Merge to keep only new leads**  
    - *Type & Role:* Merge node configured to combine two inputs with a "keepNonMatches" join mode, meaning it outputs records from the first input (Eventbrite attendees) that do not have a matching email in the second input (Pipedrive leads).  
    - *Configuration Highlights:*  
      - Uses "email" from Eventbrite and "primary_email" from Pipedrive as matching keys.  
      - Outputs only unmatched items from input1 (Eventbrite data).  
    - *Input/Output:*  
      - Input 1: Eventbrite attendees  
      - Input 2: Pipedrive persons  
      - Output: New leads only (attendees not found in Pipedrive).  
    - *Failures & Edge Cases:*  
      - If emails are missing or inconsistent, duplicates may be created or leads missed.  
      - Case sensitivity or formatting differences in emails may affect matching accuracy.  
    - *Version Requirements:* Merge node version 3.1 or higher for this join mode.  
    - *Sub-workflow:* None.

---

#### 1.5 Lead Creation in Pipedrive

- **Overview:**  
  Creates new leads in Pipedrive for attendees identified as not existing in the CRM, mapping Eventbrite data fields to Pipedrive person properties including custom fields.

- **Nodes Involved:**  
  - Add New Leads to Pipedrive

- **Node Details:**

  - **Add New Leads to Pipedrive**  
    - *Type & Role:* Pipedrive node performing "create" operation on "person" resource, adding new contacts.  
    - *Configuration Highlights:*  
      - Maps attendee JSON fields to Pipedrive person fields: name, email, work phone.  
      - Sets custom properties via Pipedrive custom field IDs; these must be updated to match target Pipedrive account.  
      - Uses Pipedrive API credentials.  
      - Executes once per item (not batch).  
    - *Key Expressions:*  
      - Name: `={{ $json.name }}`  
      - Email: `={{ $json.email }}`  
      - Phone: `={{ $json.work_phone }}`  
      - Custom Properties: IDs mapped to values like company and event date from attendee data.  
    - *Input/Output:*  
      - Input: Filtered new leads from Merge node  
      - Output: Created Pipedrive person records.  
    - *Failures & Edge Cases:*  
      - Invalid or missing required fields may cause create failures.  
      - Incorrect custom field IDs will prevent data mapping.  
      - API errors or rate limits can cause failures.  
    - *Version Requirements:* Compatible with Pipedrive node v1+.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                    | Node Type         | Functional Role                       | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                                                        |
|------------------------------|-------------------|------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Daily               | Schedule Trigger  | Initiates workflow on schedule     | —                                | Extract Eventbrite Signups, Extract current leads in pipedrive | Default interval is every 10 minutes; adjustable.                                                                                                |
| Test or Manually run workflow| Manual Trigger    | Allows manual workflow execution   | —                                | Extract Eventbrite Signups, Extract current leads in pipedrive | Useful for testing and manual syncs.                                                                                                            |
| Extract Eventbrite Signups   | Code              | Fetches events and attendees from Eventbrite | Schedule Daily, Test or Manually run workflow | Merge to keep only new leads       | Replace token and org ID; customize attendee fields; handles pagination.                                                                         |
| Extract current leads in pipedrive | Pipedrive        | Retrieves existing leads from Pipedrive | Schedule Daily, Test or Manually run workflow | Merge to keep only new leads       | Requires Pipedrive API token; fetches all persons for deduplication.                                                                             |
| Merge to keep only new leads | Merge             | Filters Eventbrite attendees to new leads | Extract Eventbrite Signups, Extract current leads in pipedrive | Add New Leads to Pipedrive         | Matches on email vs primary_email; outputs only unmatched attendees.                                                                             |
| Add New Leads to Pipedrive  | Pipedrive         | Creates new leads in Pipedrive     | Merge to keep only new leads      | —                              | Map fields and custom fields; update custom field IDs; requires Pipedrive API token.                                                             |
| Sticky Note                 | Sticky Note       | Workflow overview and contact info | —                                | —                              | "Eventbrite → Pipedrive Lead‑Sync Workflow\nNeed help? Email rbreen@ynteractive.com"                                                             |
| Sticky Note1                | Sticky Note       | Setup instructions and help links  | —                                | —                              | Detailed configuration guide, mapping, schedule setup, and contact links (email, website, YouTube, LinkedIn).                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Schedule Daily**  
   - Add a Schedule Trigger node.  
   - Set interval to "minutes" with a value of 10 (or desired frequency).  
   - No credentials required.

2. **Create Trigger Node: Test or Manually run workflow**  
   - Add a Manual Trigger node for manual execution/testing.

3. **Create Code Node: Extract Eventbrite Signups**  
   - Add a Code node (JavaScript).  
   - Paste the provided JavaScript code for fetching Eventbrite events and attendees.  
   - Replace placeholder values:  
     - `token` → Your Eventbrite personal OAuth token.  
     - `orgId` → Your Eventbrite organization ID.  
   - Customize the attendee fields inside the push to `allAttendees` as needed.  
   - No credentials needed; code uses HTTP requests.

4. **Create Pipedrive Node: Extract current leads in pipedrive**  
   - Add a Pipedrive node.  
   - Set resource to "person", operation to "getAll".  
   - Check "Return All" to fetch all leads.  
   - Configure credentials with your Pipedrive API token.

5. **Create Merge Node: Merge to keep only new leads**  
   - Add a Merge node.  
   - Set mode to "Combine".  
   - Enable "Advanced" options.  
   - Set Join Mode to "Keep Non Matches" (keeps items in input1 that do not match input2).  
   - Set merge fields: Input1 field `email` and Input2 field `primary_email`.

6. **Create Pipedrive Node: Add New Leads to Pipedrive**  
   - Add a Pipedrive node.  
   - Set resource to "person", operation to "create".  
   - Map fields:  
     - Name: `={{ $json.name }}`  
     - Email: `={{ $json.email }}` (as array)  
     - Phone: `={{ $json.work_phone }}` (as array)  
     - Custom Properties: Map desired custom fields by their Pipedrive custom field IDs.  
   - Use your Pipedrive API credentials.

7. **Connect Nodes:**  
   - Connect "Schedule Daily" main output to both "Extract Eventbrite Signups" and "Extract current leads in pipedrive".  
   - Connect "Test or Manually run workflow" similarly to these two nodes.  
   - Connect outputs of "Extract Eventbrite Signups" and "Extract current leads in pipedrive" to inputs 1 and 2 respectively of "Merge to keep only new leads".  
   - Connect output of "Merge to keep only new leads" to "Add New Leads to Pipedrive".

8. **Add Sticky Notes for Documentation (Optional):**  
   - Add sticky notes containing overview, setup instructions, and contact info for ease of maintenance.

9. **Finalize and Test:**  
   - Save the workflow.  
   - Run manually to verify correct operation and check for duplicates.  
   - Activate the workflow to enable scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Need help? Email rbreen@ynteractive.com                                                                                          | Support contact                                             |
| Workflow setup instructions include importing JSON, updating API tokens, customizing fields, and scheduling intervals.           | Setup guidance in sticky notes                              |
| Useful links for support and training: https://ynteractive.com, https://www.youtube.com/@ynteractivetraining, LinkedIn profile: https://www.linkedin.com/in/robertbreen | External resources and training                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.