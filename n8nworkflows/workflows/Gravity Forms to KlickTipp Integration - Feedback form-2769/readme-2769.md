Gravity Forms to KlickTipp Integration - Feedback form

https://n8nworkflows.xyz/workflows/gravity-forms-to-klicktipp-integration---feedback-form-2769


# Gravity Forms to KlickTipp Integration - Feedback form

### 1. Workflow Overview

This workflow automates the integration of customer feedback submitted via Gravity Forms into KlickTipp, a marketing automation platform. It captures new form submissions, transforms and validates the data to meet KlickTipp’s API requirements, manages subscriber creation or updates, and handles tagging of contacts based on form responses.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures new Gravity Forms submissions via a webhook.
- **1.2 Data Transformation:** Processes and formats raw form data (phone numbers, dates, ratings) for KlickTipp compatibility.
- **1.3 Subscriber Management:** Adds or updates contacts in KlickTipp subscriber lists with mapped custom fields.
- **1.4 Tag Management:** Extracts, checks, creates (if needed), and applies tags to contacts in KlickTipp.
- **1.5 Error Handling and Workflow Control:** Ensures smooth branching and error resilience during tag creation and contact tagging.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow on receiving a new form submission from Gravity Forms via a webhook.

- **Nodes Involved:**  
  - New submission via Gravityforms (Webhook)

- **Node Details:**  

  - **New submission via Gravityforms**  
    - Type: Webhook  
    - Role: Entry point capturing POST requests from Gravity Forms submissions.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: Unique webhook ID path (`9e8feb6b-df09-4f17-baf0-9fa3b8c0093c`)  
    - Inputs: External HTTP POST from Gravity Forms  
    - Outputs: JSON payload containing form fields indexed by field IDs (e.g., `body['4']` for email)  
    - Edge Cases:  
      - Missing or malformed payloads could cause downstream errors if not handled.  
      - Requires Gravity Forms to be configured to send submissions to this webhook URL.  
    - Notes: This node is critical as the workflow’s trigger.

#### 1.2 Data Transformation

- **Overview:**  
  Converts and validates raw Gravity Forms data into formats compatible with KlickTipp API, including phone number normalization, date conversion, and scaling numeric ratings.

- **Nodes Involved:**  
  - Convert and set feedback data (Set)

- **Node Details:**  

  - **Convert and set feedback data**  
    - Type: Set  
    - Role: Data transformation and enrichment node.  
    - Configuration:  
      - Assigns new fields:  
        - `mobile_number`: Converts phone number to numeric-only format with international prefix "00" replacing "+"  
        - `birthday`: Converts birthday string to UNIX timestamp (seconds)  
        - `webinar_choice`: Parses webinar date/time string from "DD.MM.YYYY HH:mm" to UNIX timestamp  
        - `webinar_rating`: Multiplies decimal rating by 100 for KlickTipp format  
    - Expressions: Uses JavaScript expressions to perform regex replacements and date parsing.  
    - Inputs: JSON from webhook node (`body` field)  
    - Outputs: JSON enriched with transformed fields (`mobile_number`, `birthday`, `webinar_choice`, `webinar_rating`)  
    - Edge Cases:  
      - Invalid or missing phone numbers result in empty string `''`  
      - Date parsing may fail if input format deviates, potentially resulting in `NaN` timestamps  
      - JSON parsing for webinar selection is carefully handled to avoid exceptions  
    - Notes: Ensures data integrity before API submission.

#### 1.3 Subscriber Management

- **Overview:**  
  Adds or updates a subscriber in KlickTipp using the transformed data, mapping form fields to KlickTipp custom fields.

- **Nodes Involved:**  
  - Subscribe contact in KlickTipp (KlickTipp node)

- **Node Details:**  

  - **Subscribe contact in KlickTipp**  
    - Type: KlickTipp (community node)  
    - Role: Subscribes or updates a contact in a specific KlickTipp list.  
    - Configuration:  
      - Email: Extracted from Gravity Forms field `body['4']`  
      - SMS Number: Uses transformed `mobile_number`  
      - Custom Fields: Maps multiple Gravity Forms fields and transformed data to KlickTipp fields, e.g., first name, last name, birthday, webinar rating, feedback, webinar choice.  
      - List ID: Fixed KlickTipp subscriber list ID `358895`  
    - Inputs: Transformed JSON from Set node  
    - Outputs: KlickTipp API response confirming subscription  
    - Credentials: KlickTipp API credentials required  
    - Edge Cases:  
      - API authentication errors  
      - Invalid email or missing required fields may cause subscription failure  
      - Network timeouts or KlickTipp service unavailability  
    - Notes: This node is central to syncing contacts.

#### 1.4 Tag Management

- **Overview:**  
  Extracts tags from form data, checks which tags already exist in KlickTipp, creates missing tags, then applies all relevant tags to the subscriber.

- **Nodes Involved:**  
  - Define Array of tags from Gravityforms (Set)  
  - Split Out Gravityforms tags (SplitOut)  
  - Get list of all existing tags (KlickTipp)  
  - Merge (Merge)  
  - Tag creation check (If)  
  - Create the tag in KlickTipp (KlickTipp)  
  - Aggregate array of created tags (Aggregate)  
  - Aggregate tags to add to contact (Aggregate)  
  - Tag contact directly in KlickTipp (KlickTipp)  
  - Tag contact KlickTipp after tag creation (KlickTipp)

- **Node Details:**  

  - **Define Array of tags from Gravityforms**  
    - Type: Set  
    - Role: Constructs an array of tags from various form fields (e.g., webinar rating, webinar date, reminder interval)  
    - Configuration: Uses JavaScript to safely parse and flatten tag arrays, filtering out nulls  
    - Inputs: Original Gravity Forms JSON  
    - Outputs: JSON with `tags` array field  
    - Edge Cases: JSON parsing errors handled with try-catch, fallback to null  

  - **Split Out Gravityforms tags**  
    - Type: SplitOut  
    - Role: Splits the `tags` array into individual items for processing  
    - Inputs: JSON with `tags` array  
    - Outputs: Multiple items each containing a single tag string  

  - **Get list of all existing tags**  
    - Type: KlickTipp  
    - Role: Retrieves all tags currently existing in KlickTipp for comparison  
    - Inputs: Triggered after subscriber subscription  
    - Outputs: List of existing tags with IDs  

  - **Merge**  
    - Type: Merge  
    - Role: Performs a SQL-style left join between tags from Gravity Forms and existing KlickTipp tags to identify which tags need creation  
    - Inputs:  
      - Input1: Tags from Gravity Forms (split out)  
      - Input2: Existing KlickTipp tags  
    - Outputs: Items annotated with `exist` boolean and `tag_id` if existing  

  - **Tag creation check**  
    - Type: If  
    - Role: Branches workflow based on whether tags exist (`exist == true`) or not  
    - Outputs:  
      - True branch: Tags exist, proceed to aggregate existing tag IDs  
      - False branch: Tags missing, proceed to create tags  

  - **Create the tag in KlickTipp**  
    - Type: KlickTipp  
    - Role: Creates new tags in KlickTipp for missing tags  
    - Inputs: Tag names from false branch of If node  
    - Outputs: Newly created tag IDs  

  - **Aggregate array of created tags**  
    - Type: Aggregate  
    - Role: Aggregates newly created tag IDs into a list for tagging contacts  

  - **Aggregate tags to add to contact**  
    - Type: Aggregate  
    - Role: Aggregates existing tag IDs into a list for tagging contacts  

  - **Tag contact directly in KlickTipp**  
    - Type: KlickTipp  
    - Role: Applies existing tags to subscriber contacts  
    - Inputs: Email and aggregated tag IDs  

  - **Tag contact KlickTipp after tag creation**  
    - Type: KlickTipp  
    - Role: Applies newly created tags to subscriber contacts after creation  
    - Inputs: Email and aggregated new tag IDs  

- **Edge Cases:**  
  - Tag creation may fail due to API limits or duplicate names  
  - Tag aggregation may produce empty lists if no tags exist or created  
  - Tag application may fail if subscriber email is invalid or missing  
  - Merge SQL query depends on correct data structure; malformed inputs may cause errors  

#### 1.5 Error Handling and Workflow Control

- **Overview:**  
  The workflow uses conditional branching and aggregation nodes to ensure that tags are created only if missing and applied correctly, preventing redundant API calls and errors.

- **Nodes Involved:**  
  - Tag creation check (If)  
  - Merge (Merge)  
  - Aggregate nodes  

- **Node Details:**  
  - The If node `Tag creation check` directs the flow to either create missing tags or proceed with tagging existing ones.  
  - Aggregation nodes consolidate tag IDs for batch tagging operations.  
  - This design minimizes API calls and handles empty or invalid tag arrays gracefully.  

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                                | Input Node(s)                     | Output Node(s)                                  | Sticky Note                                                                                                           |
|-------------------------------|-----------------------|-----------------------------------------------|----------------------------------|------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| New submission via Gravityforms | Webhook               | Captures Gravity Forms form submissions       | -                                | Convert and set feedback data                   | This webhook node captures incoming data from the Gravity Forms plugin on the website. It triggers the workflow when a new form submission is received. |
| Convert and set feedback data  | Set                   | Transforms and formats form data for KlickTipp | New submission via Gravityforms  | Subscribe contact in KlickTipp                   | This node transforms the form data from Gravity Forms into the appropriate format required for the KlickTipp API.      |
| Subscribe contact in KlickTipp | KlickTipp             | Subscribes/updates contact in KlickTipp list  | Convert and set feedback data     | Define Array of tags from Gravityforms, Get list of all existing tags | This node subscribes the formatted contact data to a specific KlickTipp list.                                         |
| Define Array of tags from Gravityforms | Set           | Defines tags array from form submission        | Subscribe contact in KlickTipp    | Split Out Gravityforms tags                      | This node defines tags based on the form submission, such as the webinar selection, date, and reminder interval.       |
| Split Out Gravityforms tags    | SplitOut              | Splits tags array into individual tag items   | Define Array of tags from Gravityforms | Merge                                          | In this node we split the created array again into items so we can merge them with the existing tags we request from KlickTipp. |
| Get list of all existing tags  | KlickTipp             | Fetches all existing tags in KlickTipp         | Subscribe contact in KlickTipp    | Merge                                           | This node fetches all tags that already exist in KlickTipp.                                                           |
| Merge                         | Merge                 | Joins form tags with existing KlickTipp tags  | Split Out Gravityforms tags, Get list of all existing tags | Tag creation check                             | This node merges the tags which are fetched via the form with the existing tags we requested in order to identify if new tags need to be created. |
| Tag creation check             | If                    | Checks if tags exist, branches workflow        | Merge                           | Aggregate tags to add to contact (true), Create the tag in KlickTipp (false) | This node checks the result of the tag comparison and branches the workflow accordingly in order to directly tag the contact or to create the tag first and then follow through with the tagging. |
| Aggregate tags to add to contact | Aggregate           | Aggregates existing tag IDs for tagging        | Tag creation check (true branch) | Tag contact directly in KlickTipp               | This node aggregates all IDs of the existing tags to a list.                                                           |
| Create the tag in KlickTipp    | KlickTipp             | Creates new tags in KlickTipp                   | Tag creation check (false branch) | Aggregate array of created tags                  | Creates a new tag in KlickTipp if it does not already exist.                                                           |
| Aggregate array of created tags | Aggregate            | Aggregates newly created tag IDs                | Create the tag in KlickTipp       | Tag contact KlickTipp after tag creation         | This node aggregates all IDs of the newly created tags to a list.                                                      |
| Tag contact directly in KlickTipp | KlickTipp          | Applies existing tags to subscriber             | Aggregate tags to add to contact  | -                                              | Applies existing tags to a subscriber in KlickTipp.                                                                    |
| Tag contact KlickTipp after tag creation | KlickTipp     | Applies newly created tags to subscriber        | Aggregate array of created tags   | -                                              | Associates a specific tag with a subscriber in KlickTipp using their email address.                                     |
| Sticky Note1                  | StickyNote             | Documentation and overview                       | -                                | -                                              | See detailed workflow introduction, benefits, key features, setup instructions, and testing notes.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `New submission via Gravityforms`  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `9e8feb6b-df09-4f17-baf0-9fa3b8c0093c`)  
   - Purpose: Receive Gravity Forms submissions  

2. **Create Set Node for Data Transformation**  
   - Type: Set  
   - Name: `Convert and set feedback data`  
   - Assign fields:  
     - `mobile_number`: Use expression to convert phone number to numeric-only with "00" prefix replacing "+"  
       ```js
       $json.body['5'] ? $json.body['5'].replace(/^\+/, '00').replace(/[^0-9]/g, '') : ''
       ```  
     - `birthday`: Convert birthday string to UNIX timestamp (seconds)  
       ```js
       Math.floor(new Date($json.body['6'] + 'T00:00:00').getTime() / 1000)
       ```  
     - `webinar_choice`: Convert webinar date/time from "DD.MM.YYYY HH:mm" to UNIX timestamp  
       ```js
       Math.floor(new Date($json.body['13'].replace(/(\d{2})\.(\d{2})\.(\d{4})/, "$2/$1/$3")).getTime() / 1000)
       ```  
     - `webinar_rating`: Multiply decimal rating by 100  
       ```js
       $json.body['8'] * 100
       ```  
   - Connect output of webhook node to this node  

3. **Create KlickTipp Node to Subscribe Contact**  
   - Type: KlickTipp (community node)  
   - Name: `Subscribe contact in KlickTipp`  
   - Operation: `subscribe`  
   - Resource: `subscriber`  
   - List ID: `358895` (adjust to your KlickTipp list)  
   - Email: `={{ $('New submission via Gravityforms').item.json.body['4'] }}`  
   - SMS Number: `={{ $json.mobile_number }}`  
   - Map custom fields with expressions referencing Gravity Forms fields and transformed data, e.g.:  
     - First Name: `={{ $('New submission via Gravityforms').item.json.body['1'] }}`  
     - Last Name: `={{ $('New submission via Gravityforms').item.json.body['3'] }}`  
     - Birthday: `={{ $json.birthday }}`  
     - Webinar Rating: `={{ $json.webinar_rating }}`  
     - Webinar Choice: `={{ $json.webinar_choice }}`  
   - Connect output of Set node to this node  
   - Configure KlickTipp API credentials  

4. **Create Set Node to Define Tags Array**  
   - Type: Set  
   - Name: `Define Array of tags from Gravityforms`  
   - Assign field `tags` as an array combining:  
     - Webinar rating (field '8')  
     - Webinar date (field '13')  
     - Parsed JSON from field '11' (handle parsing errors gracefully)  
   - Use JavaScript expression to flatten and filter nulls  
   - Connect output of KlickTipp subscribe node to this node  

5. **Create SplitOut Node to Split Tags Array**  
   - Type: SplitOut  
   - Name: `Split Out Gravityforms tags`  
   - Field to split out: `tags`  
   - Connect output of previous Set node to this node  

6. **Create KlickTipp Node to Get Existing Tags**  
   - Type: KlickTipp  
   - Name: `Get list of all existing tags`  
   - Operation: fetch tags (default)  
   - Connect output of KlickTipp subscribe node to this node  

7. **Create Merge Node to Join Tags**  
   - Type: Merge  
   - Name: `Merge`  
   - Mode: `combineBySql`  
   - SQL Query:  
     ```sql
     SELECT 
       input1.tags AS name,
       IF(input2.value IS NOT NULL, true, false) AS exist,
       input2.id AS tag_id
     FROM 
       input1
     LEFT JOIN 
       input2 
     ON 
       input1.tags = input2.value
     ```  
   - Connect outputs:  
     - Input1: SplitOut node (tags from form)  
     - Input2: Get existing tags node  

8. **Create If Node to Check Tag Existence**  
   - Type: If  
   - Name: `Tag creation check`  
   - Condition: Check if `$json.exist === true`  
   - Connect output of Merge node to this node  

9. **Create KlickTipp Node to Create Tags**  
   - Type: KlickTipp  
   - Name: `Create the tag in KlickTipp`  
   - Operation: `create`  
   - Name: `={{ $json.name }}`  
   - Connect false branch of If node to this node  

10. **Create Aggregate Node for Created Tags**  
    - Type: Aggregate  
    - Name: `Aggregate array of created tags`  
    - Aggregate field: `id` renamed to `tag_ids`  
    - Connect output of Create tag node to this node  

11. **Create Aggregate Node for Existing Tags**  
    - Type: Aggregate  
    - Name: `Aggregate tags to add to contact`  
    - Aggregate field: `tag_id` renamed to `tag_ids`  
    - Connect true branch of If node to this node  

12. **Create KlickTipp Node to Tag Contact with Existing Tags**  
    - Type: KlickTipp  
    - Name: `Tag contact directly in KlickTipp`  
    - Operation: `contact-tagging`  
    - Email: `={{ $('New submission via Gravityforms').item.json.body['4'] }}`  
    - Tag IDs: `={{ $json.tag_ids }}`  
    - Connect output of Aggregate existing tags node to this node  

13. **Create KlickTipp Node to Tag Contact After Tag Creation**  
    - Type: KlickTipp  
    - Name: `Tag contact KlickTipp after tag creation`  
    - Operation: `contact-tagging`  
    - Email: `={{ $('New submission via Gravityforms').item.json.body['4'] }}`  
    - Tag IDs: `={{ $json.tag_ids }}`  
    - Connect output of Aggregate created tags node to this node  

14. **Connect Nodes to Complete Tagging Flow**  
    - Connect output of KlickTipp subscribe node to both:  
      - Define Array of tags from Gravityforms  
      - Get list of all existing tags  

15. **Configure Credentials**  
    - KlickTipp API credentials must be set for all KlickTipp nodes  
    - Gravity Forms webhook must be configured on the Gravity Forms side to POST submissions to the webhook URL  

16. **Testing**  
    - Submit test forms in Gravity Forms  
    - Verify contacts and tags in KlickTipp  
    - Test edge cases: missing phone, invalid dates, empty tags  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow relies on a community node for KlickTipp, which requires a self-hosted n8n environment.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Community node disclaimer                                                                                  |
| After creating custom fields in KlickTipp (e.g., Gravityforms_URL_Linkedin, Gravityforms_kurs/webinar_zeitpunkt), allow 10-15 minutes for synchronization. If fields do not appear, reconnect KlickTipp credentials.                                                                                                                                                                                                                                                                                                                                                                                                | Setup instructions                                                                                         |
| The workflow automates lead generation by importing contacts directly into KlickTipp, enabling immediate use in campaigns and automations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Benefits                                                                                                  |
| For detailed field mapping, verify and customize the KlickTipp node field assignments to match your specific Gravity Forms setup and KlickTipp list.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Customization note                                                                                         |
| Testing should include submitting forms and verifying data in KlickTipp, including simulating missing or invalid data to ensure error handling robustness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Testing and deployment                                                                                     |
| ![Source example](https://mail.cdndata.io/user/images/kt1073234/share_link_GravityForms_fields.png#full-width)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Example of Gravity Forms field mapping                                                                    |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and modify the Gravity Forms to KlickTipp integration workflow with confidence.