Jotform to KlickTipp Integration - Webinar registration

https://n8nworkflows.xyz/workflows/jotform-to-klicktipp-integration---webinar-registration-2755


# Jotform to KlickTipp Integration - Webinar registration

### 1. Workflow Overview

This workflow automates the integration of webinar registration data submitted via JotForm into KlickTipp, a marketing automation platform. It captures new webinar registrations, validates and transforms the data to meet KlickTipp’s API requirements, manages subscriber information, and handles dynamic tagging for segmented marketing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new webinar registration submissions from JotForm.
- **1.2 Data Transformation:** Validates and formats raw form data (phone numbers, dates, URLs, numerical fields) for KlickTipp compatibility.
- **1.3 Subscriber Management:** Adds or updates the registrant as a subscriber in KlickTipp with mapped custom fields.
- **1.4 Tag Management:** Dynamically creates and applies tags based on registration details to enable targeted marketing automation.
- **1.5 Error Handling & Validation:** Embedded within transformation and tagging logic to ensure data integrity and prevent invalid submissions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow upon a new webinar registration submitted via JotForm, capturing all participant details and preferences.

**Nodes Involved:**  
- New webinar booking via JotForm

**Node Details:**

- **New webinar booking via JotForm**  
  - *Type:* JotForm Trigger  
  - *Role:* Initiates workflow on new form submission.  
  - *Configuration:* Connected to a specific JotForm form ID (250054687472360).  
  - *Inputs:* Webhook triggered by JotForm submission.  
  - *Outputs:* Emits raw form data JSON for downstream processing.  
  - *Edge Cases:* Possible webhook failures if JotForm credentials expire or network issues occur.  
  - *Notes:* Requires JotForm API credentials configured in n8n.

---

#### 1.2 Data Transformation

**Overview:**  
Processes raw submission data to standardize and validate fields such as phone numbers, birthdates, LinkedIn URLs, work experience, and webinar start times, ensuring compatibility with KlickTipp’s API.

**Nodes Involved:**  
- Convert and set webinar data

**Node Details:**

- **Convert and set webinar data**  
  - *Type:* Set Node  
  - *Role:* Transforms and validates input data fields.  
  - *Key Transformations:*  
    - Phone number: Converts to numeric-only format with international prefix "00".  
    - Birthday: Converts date components into UNIX timestamp (seconds).  
    - LinkedIn URL: Validates URL format; applies fallback URL if invalid.  
    - Work experience: Multiplies decimal years by 100 for scaling.  
    - Webinar start date/time: Parses and converts to UNIX timestamp considering German timezone.  
  - *Inputs:* Raw form submission JSON from JotForm trigger.  
  - *Outputs:* JSON with transformed fields for KlickTipp consumption.  
  - *Edge Cases:*  
    - Missing or malformed date/time fields return empty strings.  
    - Invalid LinkedIn URLs replaced with fallback URL.  
    - Phone number formatting assumes presence of "Mobilrufnummer.full" field.  
  - *Version:* Uses n8n Set node version 3.4 with JavaScript expressions.

---

#### 1.3 Subscriber Management

**Overview:**  
Adds or updates the webinar participant as a subscriber in KlickTipp, mapping all relevant personal and webinar-specific data fields.

**Nodes Involved:**  
- Subscribe contact in KlickTipp

**Node Details:**

- **Subscribe contact in KlickTipp**  
  - *Type:* KlickTipp Node (Subscriber)  
  - *Role:* Subscribes or updates contact in KlickTipp subscriber list.  
  - *Configuration:*  
    - Email sourced from JotForm submission.  
    - Custom fields mapped: first name, last name, birthday (UNIX timestamp), LinkedIn URL, scaled work experience, webinar start timestamp, questions/notes, webinar choice, reminder interval.  
    - SMS number set from transformed mobile number.  
    - List ID specified (358895).  
  - *Inputs:* Transformed data from "Convert and set webinar data" node.  
  - *Outputs:* Confirmation or error response from KlickTipp API.  
  - *Edge Cases:*  
    - API authentication failures if KlickTipp credentials expire.  
    - Data mapping errors if expected fields are missing.  
  - *Credentials:* Requires KlickTipp API credentials.

---

#### 1.4 Tag Management

**Overview:**  
Manages dynamic tagging of contacts in KlickTipp based on webinar registration details to facilitate targeted marketing automation.

**Nodes Involved:**  
- Define Array of tags from Jotform  
- Split Out Jotform tags  
- Get list of all existing tags  
- Merge  
- Tag creation check (If node)  
- Create the tag in KlickTipp  
- Aggregate array of created tags  
- Aggregate tags to add to contact  
- Tag contact directly in KlickTipp  
- Tag contact KlickTipp after tag creation

**Node Details:**

- **Define Array of tags from Jotform**  
  - *Type:* Set Node  
  - *Role:* Creates an array of tags from form submission fields: webinar choice, date, and reminder interval.  
  - *Inputs:* JotForm submission JSON.  
  - *Outputs:* JSON array of tags.  
  - *Edge Cases:* Empty or missing tag fields result in empty or incomplete arrays.

- **Split Out Jotform tags**  
  - *Type:* SplitOut Node  
  - *Role:* Splits the tags array into individual items for processing.  
  - *Inputs:* Tags array from previous node.  
  - *Outputs:* Individual tag items.  

- **Get list of all existing tags**  
  - *Type:* KlickTipp Node (Tag Retrieval)  
  - *Role:* Fetches all existing tags from KlickTipp to check for duplicates.  
  - *Inputs:* Triggered after subscription node.  
  - *Outputs:* List of existing tags with IDs.  

- **Merge**  
  - *Type:* Merge Node (SQL Mode)  
  - *Role:* Joins form-submitted tags with existing KlickTipp tags to identify which tags exist and which are new.  
  - *Inputs:*  
    - Input1: Tags from form submission.  
    - Input2: Existing tags from KlickTipp.  
  - *Outputs:* Tag existence boolean and tag IDs.  

- **Tag creation check**  
  - *Type:* If Node  
  - *Role:* Branches workflow based on whether tags exist.  
  - *Condition:* Checks if tag exists (boolean).  
  - *Outputs:*  
    - True branch: Proceed to aggregate existing tag IDs.  
    - False branch: Create new tag in KlickTipp.  

- **Create the tag in KlickTipp**  
  - *Type:* KlickTipp Node (Tag Creation)  
  - *Role:* Creates new tags in KlickTipp for those not found.  
  - *Inputs:* Tag names from merge node.  
  - *Outputs:* Newly created tag IDs.  

- **Aggregate array of created tags**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates IDs of newly created tags into a list.  

- **Aggregate tags to add to contact**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates IDs of existing tags to a list.  

- **Tag contact directly in KlickTipp**  
  - *Type:* KlickTipp Node (Contact Tagging)  
  - *Role:* Applies existing tags to subscriber using email and tag IDs.  

- **Tag contact KlickTipp after tag creation**  
  - *Type:* KlickTipp Node (Contact Tagging)  
  - *Role:* Applies newly created tags to subscriber.  

**Edge Cases & Failure Modes:**  
- API rate limits or authentication failures on KlickTipp nodes.  
- Tag creation conflicts or invalid tag names.  
- Merge node SQL query errors if input data is malformed.  
- If node branching depends on accurate boolean evaluation; expression errors may cause misrouting.

---

#### 1.5 Error Handling & Validation

**Overview:**  
Embedded within data transformation and tagging logic to ensure only valid, correctly formatted data is submitted to KlickTipp, preventing errors downstream.

**Nodes Involved:**  
- Convert and set webinar data (validation expressions)  
- Tag creation check (conditional branching)  

**Details:**  
- Validation of phone numbers, URLs, and dates is performed via JavaScript expressions in the Set node.  
- Invalid URLs are replaced with a fallback URL to avoid API errors.  
- Date parsing includes format checks and fallback to empty strings if invalid.  
- Conditional branching ensures new tags are only created if they do not already exist.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                  | Input Node(s)                         | Output Node(s)                                   | Sticky Note                                                                                                                                    |
|-----------------------------------|----------------------------|-------------------------------------------------|-------------------------------------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| New webinar booking via JotForm    | JotForm Trigger            | Captures new webinar registration submissions   | —                                   | Convert and set webinar data                      | Triggers the workflow when a new form submission is received on JotForm.                                                                        |
| Convert and set webinar data       | Set                        | Transforms and validates raw form data           | New webinar booking via JotForm      | Subscribe contact in KlickTipp                    | Formats the data received from JotForm for KlickTipp API compatibility.                                                                         |
| Subscribe contact in KlickTipp     | KlickTipp Subscriber Node  | Adds/updates subscriber in KlickTipp             | Convert and set webinar data         | Get list of all existing tags, Define Array of tags from Jotform | Adds the contact to KlickTipp using transformed webinar registration data.                                                                      |
| Get list of all existing tags      | KlickTipp Tag Retrieval    | Retrieves all existing tags from KlickTipp       | Subscribe contact in KlickTipp       | Merge                                            | Fetches all tags that already exist in KlickTipp.                                                                                              |
| Define Array of tags from Jotform  | Set                        | Creates array of tags from form submission       | Subscribe contact in KlickTipp       | Split Out Jotform tags                            | Defines tags based on form submission for further processing.                                                                                   |
| Split Out Jotform tags             | SplitOut                   | Splits tag array into individual tag items       | Define Array of tags from Jotform    | Merge                                            | Splits the created array into items to merge with existing tags.                                                                                |
| Merge                             | Merge (SQL mode)            | Joins form tags with existing KlickTipp tags     | Split Out Jotform tags, Get list of all existing tags | Tag creation check                                | Merges tags from form with existing tags to identify new tags.                                                                                 |
| Tag creation check                 | If                         | Branches based on tag existence                    | Merge                              | Aggregate tags to add to contact (true), Create the tag in KlickTipp (false) | Checks if tags exist to decide on creation or direct tagging.                                                                                   |
| Aggregate tags to add to contact   | Aggregate                  | Aggregates existing tag IDs into a list           | Tag creation check (true)            | Tag contact directly in KlickTipp                 | Aggregates all IDs of existing tags to a list.                                                                                                |
| Create the tag in KlickTipp        | KlickTipp Tag Creation     | Creates new tags in KlickTipp                      | Tag creation check (false)           | Aggregate array of created tags                   | Creates a new tag in KlickTipp if it does not already exist.                                                                                    |
| Aggregate array of created tags    | Aggregate                  | Aggregates new tag IDs into a list                 | Create the tag in KlickTipp          | Tag contact KlickTipp after tag creation          | Aggregates all IDs of newly created tags to a list.                                                                                           |
| Tag contact directly in KlickTipp  | KlickTipp Contact Tagging  | Applies existing tags to subscriber                | Aggregate tags to add to contact     | —                                                 | Applies existing tags to a subscriber in KlickTipp.                                                                                            |
| Tag contact KlickTipp after tag creation | KlickTipp Contact Tagging  | Applies newly created tags to subscriber           | Aggregate array of created tags      | —                                                 | Associates new tags with subscriber after creation.                                                                                           |
| Sticky Note1                      | Sticky Note                | Documentation and overview                         | —                                   | —                                                 | See detailed workflow introduction, benefits, key features, setup instructions, and testing notes. Includes image link for field mapping example. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node:**  
   - Type: JotForm Trigger  
   - Parameters: Select form ID `250054687472360`  
   - Credentials: Configure with valid JotForm API credentials  
   - Purpose: Trigger workflow on new webinar registration submission.

2. **Add Set Node for Data Transformation:**  
   - Name: Convert and set webinar data  
   - Type: Set  
   - Configure assignments with JavaScript expressions:  
     - `mobile_number`: Convert phone number to numeric-only with "00" prefix.  
     - `birthday`: Convert birthdate components to UNIX timestamp (seconds).  
     - `linkdein_url`: Validate LinkedIn URL format; fallback to default URL if invalid.  
     - `work_experience_in_years`: Multiply work experience decimal by 100.  
     - `webinar_start_date&time`: Parse date/time string, convert to UNIX timestamp with German timezone offset.  
   - Connect input from JotForm trigger node.

3. **Add KlickTipp Subscriber Node:**  
   - Name: Subscribe contact in KlickTipp  
   - Type: KlickTipp (Subscriber)  
   - Parameters:  
     - Email: Map from JotForm submission `Email` field.  
     - Custom fields: Map first name, last name, birthday, LinkedIn URL, work experience, webinar start timestamp, questions/notes, webinar choice, reminder interval.  
     - SMS number: Use transformed `mobile_number`.  
     - List ID: Set to `358895` (adjust as needed).  
   - Credentials: Configure with KlickTipp API credentials.  
   - Connect input from Set node.

4. **Add KlickTipp Node to Get Existing Tags:**  
   - Name: Get list of all existing tags  
   - Type: KlickTipp (Tag Retrieval)  
   - Credentials: KlickTipp API credentials  
   - Connect input from Subscribe contact node.

5. **Add Set Node to Define Tags Array:**  
   - Name: Define Array of tags from Jotform  
   - Type: Set  
   - Assign an array field `tags` containing:  
     - Webinar choice  
     - Webinar date  
     - Reminder interval  
   - Map these from JotForm submission fields.  
   - Connect input from Subscribe contact node (parallel to Get list of all existing tags).

6. **Add SplitOut Node:**  
   - Name: Split Out Jotform tags  
   - Type: SplitOut  
   - Field to split: `tags`  
   - Connect input from Define Array of tags node.

7. **Add Merge Node:**  
   - Name: Merge  
   - Type: Merge (SQL mode)  
   - Configure SQL query to left join tags from form with existing KlickTipp tags to check existence and retrieve tag IDs.  
   - Connect inputs:  
     - Input1: Split Out Jotform tags  
     - Input2: Get list of all existing tags

8. **Add If Node for Tag Creation Check:**  
   - Name: Tag creation check  
   - Type: If  
   - Condition: Check if tag exists (`$json.exist === true`)  
   - Connect input from Merge node.

9. **Add KlickTipp Node to Create Tag:**  
   - Name: Create the tag in KlickTipp  
   - Type: KlickTipp (Tag Creation)  
   - Parameters: Name mapped from tag name field.  
   - Credentials: KlickTipp API credentials  
   - Connect input from If node (false branch).

10. **Add Aggregate Node for Created Tags:**  
    - Name: Aggregate array of created tags  
    - Type: Aggregate  
    - Aggregate field: `id` renamed to `tag_ids`  
    - Connect input from Create the tag in KlickTipp node.

11. **Add Aggregate Node for Existing Tags:**  
    - Name: Aggregate tags to add to contact  
    - Type: Aggregate  
    - Aggregate field: `tag_id` renamed to `tag_ids`  
    - Connect input from If node (true branch).

12. **Add KlickTipp Node to Tag Contact (Existing Tags):**  
    - Name: Tag contact directly in KlickTipp  
    - Type: KlickTipp (Contact Tagging)  
    - Parameters:  
      - Email from JotForm submission  
      - Tag IDs from aggregated existing tags  
    - Credentials: KlickTipp API credentials  
    - Connect input from Aggregate tags to add to contact node.

13. **Add KlickTipp Node to Tag Contact (New Tags):**  
    - Name: Tag contact KlickTipp after tag creation  
    - Type: KlickTipp (Contact Tagging)  
    - Parameters:  
      - Email from JotForm submission  
      - Tag IDs from aggregated created tags  
    - Credentials: KlickTipp API credentials  
    - Connect input from Aggregate array of created tags node.

14. **Connect Subscriber Node Outputs:**  
    - Connect outputs of Subscribe contact node to both Get list of all existing tags and Define Array of tags nodes (parallel execution).

15. **Connect Aggregate Nodes and Tagging Nodes:**  
    - From Tag creation check node:  
      - True branch → Aggregate tags to add to contact → Tag contact directly in KlickTipp  
      - False branch → Create the tag in KlickTipp → Aggregate array of created tags → Tag contact KlickTipp after tag creation

16. **Add Sticky Note:**  
    - Add a sticky note node with the detailed workflow introduction, benefits, setup instructions, and testing notes for documentation purposes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow relies on a community node (KlickTipp) and is limited to self-hosted n8n environments.                                                                                                                                                                                                                                                                                                                                   | Community Node Disclaimer                                                                                         |
| Custom fields must be created in KlickTipp to match the data structure used in this workflow, including fields for LinkedIn URL, work experience, webinar start timestamp, questions/notes, webinar choice, and reminder interval.                                                                                                                                                                                                       | Setup Instructions                                                                                               |
| Field mapping should be verified and customized within KlickTipp nodes to align with your specific form and subscriber list setup.                                                                                                                                                                                                                                                                                                   | Customization note                                                                                               |
| Testing involves submitting the JotForm and verifying that data appears correctly in KlickTipp subscriber lists and tags.                                                                                                                                                                                                                                                                                                            | Testing and Deployment                                                                                            |
| Image example for field mapping: ![Source example](https://mail.cdndata.io/user/images/kt1073234/share_link_jotform_fields.png#full-width)                                                                                                                                                                                                                                                                                             | Field mapping example                                                                                            |
| Benefits include efficient lead generation, automated processes, and error-free data management, improving conversion rates and reducing manual effort.                                                                                                                                                                                                                                                                               | Benefits summary                                                                                                 |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling advanced users and automation agents to reproduce, modify, and troubleshoot the integration effectively.