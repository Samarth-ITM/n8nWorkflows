🧹 Archive (delete) duplicate items from a Notion database

https://n8nworkflows.xyz/workflows/---archive--delete--duplicate-items-from-a-notion-database-3825


# 🧹 Archive (delete) duplicate items from a Notion database

### 1. Workflow Overview

This workflow automates the identification and archiving (soft deletion) of duplicate entries in a Notion database based on a user-selected property (e.g., URL, title). It is designed for users managing large Notion databases who want to maintain data cleanliness without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Triggers:** Two optional triggers initiate the workflow — a scheduled daily run or a real-time trigger when a new page is added to the Notion database.
- **1.2 Data Retrieval and Preparation:** Retrieves all pages from the specified Notion database and formats them to isolate the property used for duplicate detection.
- **1.3 Duplicate Detection:** Aggregates all items and runs a JavaScript node to detect duplicates based on the selected property.
- **1.4 Archiving Duplicates:** Archives all detected duplicate pages except for one instance per unique property value.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers

- **Overview:**  
  Initiates the workflow either on a schedule (daily) or when a new page is added to the database, enabling flexibility in automation frequency.

- **Nodes Involved:**  
  - `Every day` (Schedule Trigger)  
  - `When a page is added to the database` (Notion Trigger)

- **Node Details:**

  - **Every day**  
    - *Type & Role:* Schedule Trigger — triggers workflow at fixed intervals.  
    - *Configuration:* Runs once every day (default interval with no additional settings).  
    - *Inputs:* None (start node).  
    - *Outputs:* Connects to `Get pages from database`.  
    - *Edge Cases:* Ensure server time zone aligns with expected schedule; possible delay if execution queue is busy.  
    - *Version:* Uses v1.2 of schedule trigger node.

  - **When a page is added to the database**  
    - *Type & Role:* Notion Trigger — polls the Notion database to detect new pages.  
    - *Configuration:* Polls every minute for new entries in the selected database.  
    - *Inputs:* None (start node).  
    - *Outputs:* Connects to `Get pages from database`.  
    - *Edge Cases:* Potential API rate limits from Notion; database ID must be correctly set; missed triggers if polling interval is too long or Notion API is down.  
    - *Version:* v1.

#### 1.2 Data Retrieval and Preparation

- **Overview:**  
  Retrieves all pages from the Notion database, formats the data to isolate the property used to identify duplicates, and aggregates all items into a single dataset for processing.

- **Nodes Involved:**  
  - `Get pages from database`  
  - `Format items properly`  
  - `Aggregate all items`

- **Node Details:**

  - **Get pages from database**  
    - *Type & Role:* Notion node — fetches all pages from the selected Notion database.  
    - *Configuration:*  
      - Operation: `getAll` database pages.  
      - `returnAll` enabled to fetch all pages without pagination limit.  
      - `databaseId` must be specified with the target database.  
    - *Inputs:* Trigger nodes (`Every day` or `When a page is added to the database`).  
    - *Outputs:* Connects to `Format items properly`.  
    - *Edge Cases:* Large databases may cause longer fetch times or API rate limits; database ID must be accurate; Notion API connectivity required.  
    - *Version:* v2.2.

  - **Format items properly**  
    - *Type & Role:* Set node — formats each page item JSON to standardize the property for duplicate checking.  
    - *Configuration:*  
      - Assigns two fields:  
        - `id`: copied directly from the input JSON's `id` field.  
        - `property_to_check`: user-defined property to detect duplicates, default placeholder `"SET YOUR PROPERTY HERE"` to be replaced with an actual property reference (recommended to use n8n’s drag-and-drop expression).  
    - *Inputs:* From `Get pages from database`.  
    - *Outputs:* Connects to `Aggregate all items`.  
    - *Edge Cases:* Failure to replace `"SET YOUR PROPERTY HERE"` results in ineffective duplicate detection; property must exist on all pages.  
    - *Version:* v3.4.

  - **Aggregate all items**  
    - *Type & Role:* Aggregate node — collects all formatted pages into one array under the field `pages`.  
    - *Configuration:* Aggregate all item data into `pages` field.  
    - *Inputs:* From `Format items properly`.  
    - *Outputs:* Connects to `Filter duplicates`.  
    - *Edge Cases:* Large datasets may affect performance; aggregation must succeed to enable proper duplicate filtering.  
    - *Version:* v1.

#### 1.3 Duplicate Detection

- **Overview:**  
  Processes the aggregated list of pages to identify duplicates based on the selected property, outputting only the duplicate entries to be archived.

- **Nodes Involved:**  
  - `Filter duplicates` (Code node)

- **Node Details:**

  - **Filter duplicates**  
    - *Type & Role:* Code node — runs custom JavaScript to detect duplicate pages.  
    - *Configuration:*  
      - JavaScript logic reads the aggregated `pages` array, tracks seen property values in a Set, and collects duplicates in a Map.  
      - Outputs an array of JSON objects where each is a duplicate (all but the first occurrence per property value).  
    - *Key expressions:*  
      - Uses `$input.first().json.pages` to access aggregated data.  
      - Iteration and conditionals to identify duplicates.  
    - *Inputs:* From `Aggregate all items`.  
    - *Outputs:* Connects to `Archive pages`.  
    - *Edge Cases:* If property values are missing or inconsistent, duplicates may be missed or falsely identified; JavaScript errors if input data structure changes; handles only one duplicate per property value (could be extended to multiple duplicates).  
    - *Version:* v2.

#### 1.4 Archiving Duplicates

- **Overview:**  
  Archives each duplicate page in Notion, effectively soft-deleting them to keep the database clean.

- **Nodes Involved:**  
  - `Archive pages` (Notion node)

- **Node Details:**

  - **Archive pages**  
    - *Type & Role:* Notion node — archives (soft deletes) the page identified by the input JSON `id`.  
    - *Configuration:*  
      - Operation: `archive`.  
      - `pageId` dynamically set using expression `={{ $json.id }}`.  
    - *Inputs:* From `Filter duplicates`.  
    - *Outputs:* None (terminal node).  
    - *Edge Cases:* Archiving may fail due to permission issues, invalid page IDs, or API limits; ensure correct authentication credentials; if a page is already archived, operation is idempotent but may generate warnings.  
    - *Version:* v2.2.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                      | Input Node(s)                   | Output Node(s)              | Sticky Note                                                                                     |
|--------------------------------|-----------------------|------------------------------------|---------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Every day                      | Schedule Trigger      | Initiates workflow daily            | None                            | Get pages from database     | ## TRIGGERS                                                                                     |
| When a page is added to the database | Notion Trigger       | Initiates workflow on new page     | None                            | Get pages from database     | ## TRIGGERS                                                                                     |
| Get pages from database        | Notion Node           | Retrieves all pages from database   | Every day, When a page is added | Format items properly       | ## GET DUPLICATE PAGES                                                                          |
| Format items properly          | Set Node              | Prepares and formats page data      | Get pages from database         | Aggregate all items         | ## GET DUPLICATE PAGES                                                                          |
| Aggregate all items            | Aggregate Node        | Aggregates all formatted pages      | Format items properly           | Filter duplicates           | ## GET DUPLICATE PAGES                                                                          |
| Filter duplicates              | Code Node             | Detects duplicate pages             | Aggregate all items             | Archive pages               | ## GET DUPLICATE PAGES                                                                          |
| Archive pages                 | Notion Node           | Archives (soft deletes) duplicates  | Filter duplicates               | None                       | ## ARCHIVE (DELETE)                                                                             |
| Sticky Note                   | Sticky Note           | Provides workflow overview & setup  | None                          | None                       | ## 🧹 Archive (delete) extra duplicate items from Notion database ... (full note content)       |
| Sticky Note1                  | Sticky Note           | Highlights trigger nodes             | None                          | None                       | ## TRIGGERS                                                                                     |
| Sticky Note2                  | Sticky Note           | Highlights duplicate detection nodes| None                          | None                       | ## GET DUPLICATE PAGES                                                                          |
| Sticky Note3                  | Sticky Note           | Highlights archiving nodes           | None                          | None                       | ## ARCHIVE (DELETE)                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers**

   - Add a **Schedule Trigger** node named `Every day`.  
     - Set interval to run once every day (default).  
     - This node will trigger the workflow daily.

   - Add a **Notion Trigger** node named `When a page is added to the database`.  
     - Configure to poll every minute.  
     - Set the database ID to your Notion target database.  
     - This node triggers the workflow whenever a new page is detected.

2. **Retrieve Pages**

   - Add a **Notion** node named `Get pages from database`.  
     - Resource: `databasePage`  
     - Operation: `getAll`  
     - Enable `returnAll` to fetch all pages.  
     - Set the `databaseId` to your Notion database ID.  
     - Connect both triggers (`Every day` and `When a page is added to the database`) to this node.

3. **Format Items**

   - Add a **Set** node named `Format items properly`.  
     - Add two fields:  
       - `id` (string): Set value to `{{ $json.id }}` (drag the id property from previous node).  
       - `property_to_check` (string): Replace default `"SET YOUR PROPERTY HERE"` with an expression referencing the property to detect duplicates (e.g., `{{ $json.properties.URL.url }}` or any other relevant property). Use n8n's expression editor for accuracy.  
     - Connect `Get pages from database` to this node.

4. **Aggregate Items**

   - Add an **Aggregate** node named `Aggregate all items`.  
     - Set the aggregation to `aggregateAllItemData`.  
     - Set `destinationFieldName` to `pages`.  
     - Connect `Format items properly` to this node.

5. **Detect Duplicates**

   - Add a **Code** node named `Filter duplicates`.  
     - Use JavaScript code to identify duplicates:  
       ```javascript
       const inputData = $input.first().json.pages;

       const seen = new Set();
       const duplicates = new Map();

       inputData.forEach(item => {
         const propertyValue = item.property_to_check;
         if (seen.has(propertyValue)) {
           duplicates.set(propertyValue, item);
         } else {
           seen.add(propertyValue);
         }
       });

       const output = Array.from(duplicates.values()).map(item => ({ json: item }));

       return output;
       ```  
     - Connect `Aggregate all items` to this node.

6. **Archive Duplicate Pages**

   - Add a **Notion** node named `Archive pages`.  
     - Operation: `archive`.  
     - Set `pageId` to `{{ $json.id }}` (from the duplicates detected).  
     - Connect `Filter duplicates` to this node.

7. **Credentials Setup**

   - For all Notion nodes (`Get pages from database`, `When a page is added to the database`, `Archive pages`), configure the Notion credentials with proper OAuth2 or integration token access that includes read and write permissions for the target database.

8. **Optional Sticky Notes**

   - Add sticky notes to document each logical block for clarity and maintenance.

9. **Activate Workflow**

   - Enable either or both triggers depending on your preference.  
   - Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow is designed to maintain Notion database cleanliness by automatically archiving extra duplicate pages based on a property.     | Workflow purpose description                                                                         |
| Replace `"SET YOUR PROPERTY HERE"` in the `Format items properly` node with the property you want to detect duplicates on (drag-and-drop recommended). | Critical setup instruction                                                                           |
| Two triggers are provided: scheduled daily runs and real-time page addition triggers, allowing flexible automation options.                 | Workflow customization and trigger details                                                          |
| Archiving duplicates is equivalent to soft-deleting them in Notion; to tag instead, replace the `Archive pages` node with an update node.   | Customization tip                                                                                     |
| Potential edge cases include API rate limits, missing or inconsistent property values, and permission issues with Notion API access.         | Error handling and robustness considerations                                                        |
| Official Notion API documentation: https://developers.notion.com/docs                                                                 | External documentation link                                                                          |
| n8n documentation for Notion nodes: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion/                              | External documentation link                                                                          |

---

This structure provides a comprehensive understanding of the workflow, enabling users or AI agents to reproduce, adapt, or troubleshoot the automation reliably.