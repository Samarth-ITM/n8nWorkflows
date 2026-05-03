Type Forms to KlickTipp Integration - Quiz

https://n8nworkflows.xyz/workflows/type-forms-to-klicktipp-integration---quiz-2774


# Type Forms to KlickTipp Integration - Quiz

### 1. Workflow Overview

This workflow automates the integration of quiz responses submitted via Typeform into the KlickTipp marketing platform. It is designed for use cases where quiz data must be captured, transformed, and synchronized as subscriber information and tags in KlickTipp, enabling streamlined lead management and targeted marketing automation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new quiz submissions from Typeform via a webhook trigger.
- **1.2 Data Transformation:** Processes and formats raw quiz data to comply with KlickTipp API requirements, including phone number normalization, date conversion, and answer mapping.
- **1.3 Subscriber Management:** Adds or updates the quiz participant as a subscriber in KlickTipp, mapping personal and quiz-related data to custom fields.
- **1.4 Tag Handling:** Extracts quiz-related tags from the submission, checks existing tags in KlickTipp, creates new tags if necessary, and associates all relevant tags with the subscriber.
- **1.5 Error Handling & Notes:** Includes mechanisms for handling empty or malformed data and provides documentation notes embedded in the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow upon receiving a new quiz submission from Typeform. It captures all form responses including personal details and quiz answers.

- **Nodes Involved:**  
  - New quiz sumbmission via Typeform

- **Node Details:**  
  - **Name:** New quiz sumbmission via Typeform  
  - **Type:** Typeform Trigger node  
  - **Role:** Listens for new submissions on a specified Typeform quiz form via webhook.  
  - **Configuration:**  
    - `formId` set to the specific Typeform quiz ID (`nRFO0o92`).  
    - Authenticated with Typeform credentials.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Emits JSON data representing the quiz submission.  
  - **Potential Failures:**  
    - Webhook misconfiguration or connectivity issues.  
    - Authentication failures with Typeform API.  
  - **Notes:** This node is the entry point of the workflow.

#### 1.2 Data Transformation

- **Overview:**  
  This block formats and converts the raw quiz data into a structure compatible with KlickTipp’s API. It normalizes phone numbers, converts birthday dates to UNIX timestamps, concatenates multiple-choice answers, and scales numeric responses.

- **Nodes Involved:**  
  - Convert and set quiz data

- **Node Details:**  
  - **Name:** Convert and set quiz data  
  - **Type:** Set node  
  - **Role:** Assigns new fields with transformed data based on expressions.  
  - **Configuration:**  
    - `mobile_number`: Converts phone numbers to numeric-only format with international prefix "00" replacing "+".  
    - `birthday`: Converts birthday string to UNIX timestamp (seconds).  
    - `question1_klicktipp_use`: Joins multiple-choice answers into a comma-separated string.  
    - `question3_amount_cht_members`: Multiplies a numeric quiz answer by 100 for scaling.  
  - **Key Expressions:**  
    - Uses JavaScript expressions to transform data fields from the incoming JSON.  
  - **Inputs:** Receives data from Typeform trigger node.  
  - **Outputs:** Emits transformed data for KlickTipp subscription.  
  - **Potential Failures:**  
    - Missing or malformed input fields causing expression errors.  
    - Date parsing errors if birthday format is invalid.  
  - **Notes:** Ensures data integrity before API submission.

#### 1.3 Subscriber Management

- **Overview:**  
  This block subscribes the participant to a KlickTipp list using the transformed data, mapping personal details and quiz answers to KlickTipp custom fields.

- **Nodes Involved:**  
  - Subscribe contact in KlickTipp

- **Node Details:**  
  - **Name:** Subscribe contact in KlickTipp  
  - **Type:** KlickTipp node (subscriber resource)  
  - **Role:** Adds or updates a subscriber in KlickTipp with mapped fields.  
  - **Configuration:**  
    - Email, first name, last name, birthday, phone number, and quiz answers mapped to KlickTipp fields.  
    - List ID specified (`358895`).  
    - Phone number passed as SMS number.  
  - **Inputs:** Receives transformed data from "Convert and set quiz data".  
  - **Outputs:** Emits subscriber confirmation data.  
  - **Potential Failures:**  
    - API authentication errors.  
    - Invalid or missing required subscriber fields.  
    - Network or timeout issues.  
  - **Notes:** Requires KlickTipp API credentials.

#### 1.4 Tag Handling

- **Overview:**  
  This block manages dynamic tagging of subscribers based on quiz responses. It extracts tags from the submission, checks existing tags in KlickTipp, creates missing tags, and applies all relevant tags to the subscriber.

- **Nodes Involved:**  
  - Define Array of tags from Typeform  
  - Split Out Typeform tags  
  - Get list of all existing tags  
  - Merge  
  - Tag creation check (If node)  
  - Create the tag in KlickTipp  
  - Aggregate array of created tags  
  - Aggregate tags to add to contact  
  - Tag contact directly in KlickTipp  
  - Tag contact KlickTipp after trag creation

- **Node Details:**  

  - **Define Array of tags from Typeform**  
    - Type: Set node  
    - Role: Creates an array of tags extracted from quiz answers (e.g., usage, company location, team size).  
    - Input: Subscriber data from "Subscribe contact in KlickTipp".  
    - Output: Array of tags for processing.  
    - Potential Failures: Empty or missing quiz answers result in empty tags array.

  - **Split Out Typeform tags**  
    - Type: SplitOut node  
    - Role: Splits the tags array into individual items for comparison.  
    - Input: Tags array from previous node.  
    - Output: Individual tag items.  

  - **Get list of all existing tags**  
    - Type: KlickTipp node  
    - Role: Retrieves all existing tags from KlickTipp to check for duplicates.  
    - Input: Triggered after subscriber subscription.  
    - Output: List of existing tags.  
    - Potential Failures: API errors or authentication issues.

  - **Merge**  
    - Type: Merge node (combineBySql mode)  
    - Role: Joins the tags from Typeform with existing KlickTipp tags to identify which tags exist and which are new.  
    - Inputs:  
      - Input1: Tags from Typeform (split out).  
      - Input2: Existing tags from KlickTipp.  
    - Output: Annotated tag list with existence flags and tag IDs.

  - **Tag creation check**  
    - Type: If node  
    - Role: Branches workflow based on whether tags exist in KlickTipp.  
    - Condition: Checks if `exist` field is true.  
    - Outputs:  
      - True branch (tag exists): proceeds to aggregate tags for direct tagging.  
      - False branch (tag missing): proceeds to create new tag.

  - **Create the tag in KlickTipp**  
    - Type: KlickTipp node  
    - Role: Creates a new tag in KlickTipp for missing tags.  
    - Input: Tag name from merge output.  
    - Output: Newly created tag ID.  
    - Potential Failures: API errors, duplicate tag creation conflicts.

  - **Aggregate array of created tags**  
    - Type: Aggregate node  
    - Role: Collects all newly created tag IDs into a list for bulk tagging.  

  - **Aggregate tags to add to contact**  
    - Type: Aggregate node  
    - Role: Collects all existing tag IDs for bulk tagging.  

  - **Tag contact directly in KlickTipp**  
    - Type: KlickTipp node  
    - Role: Applies existing tags to the subscriber immediately.  
    - Input: Subscriber email and aggregated tag IDs.  

  - **Tag contact KlickTipp after trag creation**  
    - Type: KlickTipp node  
    - Role: Applies newly created tags to the subscriber after tag creation.  
    - Input: Subscriber email and aggregated new tag IDs.  

- **Notes:**  
  - This block ensures that all relevant tags from the quiz are present in KlickTipp and associated with the subscriber.  
  - Handles dynamic tag creation and assignment.  
  - Potential failure points include API limits, authentication, and concurrency issues.

#### 1.5 Error Handling & Notes

- **Overview:**  
  The workflow includes embedded sticky notes documenting the workflow purpose, setup instructions, and benefits. It also implicitly handles empty or malformed data by using conditional expressions and default empty strings.

- **Nodes Involved:**  
  - Sticky Note1

- **Node Details:**  
  - **Name:** Sticky Note1  
  - **Type:** Sticky Note  
  - **Role:** Provides detailed documentation and instructions within the workflow canvas.  
  - **Content:**  
    - Introduction, benefits, key features, setup instructions, testing, and customization tips.  
    - Includes a visual example link for field mapping.  
  - **Notes:**  
    - Does not affect workflow execution but is crucial for maintainability and onboarding.

---

### 3. Summary Table

| Node Name                          | Node Type                      | Functional Role                        | Input Node(s)                      | Output Node(s)                             | Sticky Note                                                                                              |
|-----------------------------------|--------------------------------|-------------------------------------|----------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------|
| New quiz sumbmission via Typeform | Typeform Trigger               | Captures new quiz submissions       | None                             | Convert and set quiz data                   | Triggers the workflow when a new quiz submission is received on Type Form.                              |
| Convert and set quiz data          | Set                           | Formats and transforms quiz data    | New quiz sumbmission via Typeform | Subscribe contact in KlickTipp              | Formats the data received from the Jotform submission, ensuring it is correctly formatted for KlickTipp.|
| Subscribe contact in KlickTipp     | KlickTipp (subscriber)         | Adds/updates subscriber in KlickTipp| Convert and set quiz data        | Define Array of tags from Typeform, Get list of all existing tags | Adds the contact to KlickTipp using the transformed quiz data.                                           |
| Define Array of tags from Typeform | Set                           | Defines tags array from quiz answers| Subscribe contact in KlickTipp   | Split Out Typeform tags                      | Defines tags based on the form submission for further processing.                                       |
| Split Out Typeform tags            | SplitOut                      | Splits tags array into individual tags | Define Array of tags from Typeform | Merge                                      | Splits the created array again into items for merging with existing tags.                              |
| Get list of all existing tags     | KlickTipp (tag list)           | Fetches all existing KlickTipp tags | Subscribe contact in KlickTipp   | Merge                                       | Fetches all tags that already exist in KlickTipp.                                                       |
| Merge                            | Merge (combineBySql)            | Compares Typeform tags with existing tags | Split Out Typeform tags, Get list of all existing tags | Tag creation check                         | Merges tags to identify if new tags need creation.                                                     |
| Tag creation check                | If                            | Branches based on tag existence     | Merge                           | Aggregate tags to add to contact (true branch), Create the tag in KlickTipp (false branch) | Checks if tags exist to decide on creation or direct tagging.                                          |
| Create the tag in KlickTipp       | KlickTipp (tag creation)       | Creates new tags in KlickTipp       | Tag creation check (false branch) | Aggregate array of created tags             | Creates a new tag in KlickTipp if it does not already exist.                                           |
| Aggregate array of created tags   | Aggregate                     | Aggregates newly created tag IDs    | Create the tag in KlickTipp      | Tag contact KlickTipp after trag creation   | Aggregates all IDs of the newly created tags to a list.                                               |
| Aggregate tags to add to contact  | Aggregate                     | Aggregates existing tag IDs         | Tag creation check (true branch) | Tag contact directly in KlickTipp            | Aggregates all IDs of the existing tags to a list.                                                    |
| Tag contact directly in KlickTipp | KlickTipp (contact-tagging)    | Applies existing tags to subscriber | Aggregate tags to add to contact | None                                       | Applies existing tags to a subscriber in KlickTipp.                                                   |
| Tag contact KlickTipp after trag creation | KlickTipp (contact-tagging) | Applies newly created tags to subscriber | Aggregate array of created tags | None                                       | Associates newly created tags with the subscriber in KlickTipp.                                       |
| Sticky Note1                     | Sticky Note                   | Documentation and instructions      | None                           | None                                       | Contains detailed workflow introduction, benefits, setup instructions, and testing notes.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger Node**  
   - Type: Typeform Trigger  
   - Configure with your Typeform API credentials.  
   - Set `formId` to your quiz form ID (e.g., `nRFO0o92`).  
   - Position as the workflow entry point.

2. **Add Set Node for Data Transformation**  
   - Name: Convert and set quiz data  
   - Add assignments:  
     - `mobile_number`: Use expression to convert phone number to numeric-only with "00" prefix replacing "+".  
     - `birthday`: Convert birthday string to UNIX timestamp (seconds).  
     - `question1_klicktipp_use`: Join multiple-choice answers into comma-separated string.  
     - `question3_amount_cht_members`: Multiply numeric answer by 100.  
   - Connect output of Typeform Trigger to this node.

3. **Add KlickTipp Node to Subscribe Contact**  
   - Type: KlickTipp node, resource: subscriber, operation: subscribe  
   - Configure with KlickTipp API credentials.  
   - Set subscriber email from Typeform submission email field.  
   - Map custom fields: first name, last name, birthday (from transformed data), LinkedIn URL, quiz answers, and phone number (SMS number).  
   - Specify KlickTipp list ID for subscription.  
   - Connect output of Set node to this node.

4. **Add Set Node to Define Tags Array**  
   - Name: Define Array of tags from Typeform  
   - Assign an array field `tags` containing quiz answer values relevant for tagging (e.g., usage, company location, team size).  
   - Connect output of KlickTipp subscriber node to this node.

5. **Add SplitOut Node to Split Tags**  
   - Field to split out: `tags`  
   - Connect output of Define Array of tags node to this node.

6. **Add KlickTipp Node to Get List of Existing Tags**  
   - Type: KlickTipp node, operation: list tags  
   - Configure with KlickTipp API credentials.  
   - Connect output of KlickTipp subscriber node (parallel to Define Array of tags node) to this node.

7. **Add Merge Node (combineBySql mode)**  
   - Inputs:  
     - Input1: Output from SplitOut node (tags from Typeform).  
     - Input2: Output from Get list of existing tags node.  
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
   - Connect outputs accordingly.

8. **Add If Node to Check Tag Existence**  
   - Condition: Check if `exist` field is true.  
   - Connect Merge node output to this If node.

9. **Add KlickTipp Node to Create Tag**  
   - Operation: create tag  
   - Name: Create the tag in KlickTipp  
   - Use tag name from If node false branch.  
   - Connect If node false output to this node.

10. **Add Aggregate Node to Collect Created Tag IDs**  
    - Field to aggregate: `id` (tag ID)  
    - Rename output field to `tag_ids`.  
    - Connect output of Create tag node to this node.

11. **Add Aggregate Node to Collect Existing Tag IDs**  
    - Field to aggregate: `tag_id`  
    - Rename output field to `tag_ids`.  
    - Connect If node true output to this node.

12. **Add KlickTipp Node to Tag Contact with Existing Tags**  
    - Operation: contact-tagging  
    - Use subscriber email and aggregated existing tag IDs.  
    - Connect output of Aggregate existing tags node to this node.

13. **Add KlickTipp Node to Tag Contact After Tag Creation**  
    - Operation: contact-tagging  
    - Use subscriber email and aggregated created tag IDs.  
    - Connect output of Aggregate created tags node to this node.

14. **Add Sticky Note**  
    - Add a sticky note containing workflow documentation, setup instructions, and benefits for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow relies on a community node and is limited to self-hosted n8n environments.                                                                                                                                                         | Community node disclaimer                                                                                       |
| After creating custom fields in KlickTipp, allow 10-15 minutes for synchronization. If fields don’t appear, reconnect KlickTipp credentials.                                                                                                   | Setup instruction note                                                                                          |
| Custom fields to create in KlickTipp include: Typeform_URL_Linkedin (URL), Typeform_Frage1_klicktipp_nutzen (Text), Typeform_Frage2_klicktipp_sitz (Text), Typeform_Frage3_mitglieder_CHT (Decimal).                                               | Field setup guidance                                                                                           |
| Testing recommendation: Submit a quiz via Typeform and verify subscriber data and tags in KlickTipp.                                                                                                                                             | Testing and deployment note                                                                                    |
| Link to example field mapping image: ![Source example](https://mail.cdndata.io/user/images/kt1073234/share_link_TypeForms_fields.png#full-width)                                                                                                | Visual aid for field mapping                                                                                   |
| Benefits include efficient lead generation, automated workflows, and error-free data management.                                                                                                                                                 | Workflow benefits summary                                                                                      |
| Customize field mappings within KlickTipp nodes to match your specific account setup for accurate data synchronization.                                                                                                                         | Customization tip                                                                                              |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Type Forms to KlickTipp Integration - Quiz" workflow, ensuring clarity for advanced users and automation agents alike.