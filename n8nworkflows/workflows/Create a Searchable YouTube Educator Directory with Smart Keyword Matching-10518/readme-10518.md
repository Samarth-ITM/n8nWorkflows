Create a Searchable YouTube Educator Directory with Smart Keyword Matching

https://n8nworkflows.xyz/workflows/create-a-searchable-youtube-educator-directory-with-smart-keyword-matching-10518


# Create a Searchable YouTube Educator Directory with Smart Keyword Matching

### 1. Workflow Overview

This workflow creates a searchable directory of YouTube educational videos about n8n, allowing users to query a curated database by topic keywords. It is designed to support integration with external frontends or API clients via webhook POST requests. The workflow logically separates into two main blocks:

- **1.1 Search API**: Receives search queries through a webhook, normalizes input keywords for better matching, queries a Data Table containing video metadata, formats results with user-friendly emojis, and returns the response via the webhook.

- **1.2 Database Initialization**: A manual-triggered setup block that populates the Data Table with a predefined set of 10 curated YouTube video records. This block is intended for one-time execution before the search API is activated.

---

### 2. Block-by-Block Analysis

#### 2.1 Search API Block

**Overview:**  
This block processes incoming search requests, prepares and normalizes the search term, queries the Data Table for matching videos, formats the results into a readable message, and sends the response back to the client through the webhook.

**Nodes Involved:**  
- Webhook  
- Process Search Term (Code)  
- Get row(s) (Data Table Query)  
- Set Message  
- Respond to Webhook  

**Node Details:**

- **Webhook**  
  - *Type*: HTTP Webhook Trigger  
  - *Role*: Entry point for external POST requests with JSON body containing the search topic.  
  - *Configuration*: HTTP POST method with path `1799531d-7245-422a-b069-c76ca29bdda2`, response mode set to return data from the last node.  
  - *Inputs*: External HTTP POST request.  
  - *Outputs*: Passes received JSON to "Process Search Term".  
  - *Edge Cases*: Invalid or missing JSON body, malformed requests, slow client connections.  
  - *Version*: 2.1  

- **Process Search Term**  
  - *Type*: Code (JavaScript)  
  - *Role*: Normalizes and maps user input keywords to standardized topic tags used in the database.  
  - *Configuration*: Trims and lowercases the input topic. Applies keyword mapping rules, e.g., "talk", "voice" → "voice"; "lead" → "lead gen"; "scrape", "data" → "scraping".  
  - *Expressions*: Accesses `$json.topic` from webhook input.  
  - *Inputs*: JSON object with raw topic string.  
  - *Outputs*: Returns normalized topic in JSON for querying the Data Table.  
  - *Edge Cases*: Empty or undefined topics, unrecognized keywords default to original input (lowercased).  
  - *Version*: 2  

- **Get row(s)**  
  - *Type*: Data Table Query  
  - *Role*: Retrieves rows from the Data Table "n8n_Educator_Videos" where the "Description" column contains the normalized topic keyword (LIKE condition).  
  - *Configuration*: Filter condition uses `Description LIKE {{ $json["topic"] }}`; matchType is allConditions (single condition here).  
  - *Inputs*: Topic keyword from "Process Search Term".  
  - *Outputs*: Returns matching video records as an array under `data`.  
  - *Edge Cases*: No matches found, Data Table connectivity errors, query timeouts.  
  - *Version*: 1  

- **Set Message**  
  - *Type*: Set (Data Transformation)  
  - *Role*: Transforms the array of video records into a formatted textual message with emojis for display.  
  - *Configuration*: Maps over `$json["data"]` array to produce a string for each video, joining with double newlines; includes video title, educator, difficulty, YouTube link, and description with emojis for visual clarity.  
  - *Expressions*: Uses JavaScript map function within expression:  
    `{{ $json["data"].map(v => \`🎥 *${v["Video Title"]}* 👤 ${v["Educator"]} 🧩 Difficulty: ${v["Difficulty"]} 🔗 ${v["YouTube Link"]} 📝 ${v["Description"]}\`).join("\n\n") }}`  
  - *Inputs*: Video records array from "Get row(s)".  
  - *Outputs*: JSON with single string field "Message".  
  - *Edge Cases*: Empty arrays produce empty messages.  
  - *Version*: 3.4  

- **Respond to Webhook**  
  - *Type*: Respond to Webhook Node  
  - *Role*: Sends the formatted message back as HTTP response to the original webhook caller.  
  - *Configuration*: Default response options; uses the output of "Set Message".  
  - *Inputs*: JSON message from "Set Message".  
  - *Outputs*: HTTP Response sent to client.  
  - *Edge Cases*: Client disconnections, timeout before response.  
  - *Version*: 1.4  

---

#### 2.2 Database Initialization Block

**Overview:**  
This block, triggered manually, loads a predefined list of 10 YouTube video records into the "n8n_Educator_Videos" Data Table. It splits the records into batches to insert them one by one, ensuring the database is seeded before the search API is used.

**Nodes Involved:**  
- When clicking 'Execute workflow' (Manual Trigger)  
- Load Video Database (Code)  
- Loop Over Items (Split In Batches)  
- Insert row (Data Table Insert)  

**Node Details:**

- **When clicking 'Execute workflow'**  
  - *Type*: Manual Trigger  
  - *Role*: Starts the database seeding process manually.  
  - *Configuration*: No parameters; triggered by user action in n8n UI.  
  - *Inputs*: None.  
  - *Outputs*: Passes control to "Load Video Database".  
  - *Edge Cases*: None.  
  - *Version*: 1  

- **Load Video Database**  
  - *Type*: Code (JavaScript)  
  - *Role*: Defines an array of 10 structured video objects with metadata fields: Educator, Video Title, Difficulty, YouTube Link, Description. Converts the array into individual items for batch processing.  
  - *Configuration*: Hardcoded video dataset inside node code. Outputs array of JSON items.  
  - *Inputs*: None.  
  - *Outputs*: Array of video objects as separate items for splitting.  
  - *Edge Cases*: Static dataset; no dynamic updates.  
  - *Version*: 2  

- **Loop Over Items**  
  - *Type*: Split In Batches  
  - *Role*: Processes the array of video items one by one (batch size = 1) to insert each row separately.  
  - *Configuration*: Default batch options (batch size not explicitly set, defaults to 1).  
  - *Inputs*: Array of video items from "Load Video Database".  
  - *Outputs*: Individual video item per iteration to "Insert row".  
  - *Edge Cases*: Batch failure on single item stops loop; no retries configured.  
  - *Version*: 3  

- **Insert row**  
  - *Type*: Data Table Insert  
  - *Role*: Inserts a single video record into the "n8n_Educator_Videos" Data Table.  
  - *Configuration*: Maps JSON fields to Data Table columns: Educator, video_title, Difficulty, YouTubeLink, Description. Uses explicit field mappings with input expressions extracting each field from the incoming JSON.  
  - *Inputs*: Single video JSON item from "Loop Over Items".  
  - *Outputs*: Passes to "Load Video Database" — this output connection seems redundant or a loop to restart? (In the actual workflow, it connects back to "Load Video Database" which is unusual; likely a misconfiguration or unused).  
  - *Edge Cases*: Insert errors, Data Table connectivity issues, duplicate rows (no uniqueness enforced).  
  - *Version*: 1  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                              | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                                               |
|---------------------------|---------------------|----------------------------------------------|--------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook             | Receives POST request to start search API   | —                        | Process Search Term       | Part of Search API block: Receives POST requests with search topic, normalizes keywords, queries Data Table, returns results. |
| Process Search Term       | Code                | Normalizes input topic keywords               | Webhook                  | Get row(s)                |                                                                                                                           |
| Get row(s)                | Data Table Query    | Queries Data Table for videos matching topic | Process Search Term      | Set Message               |                                                                                                                           |
| Set Message               | Set                 | Formats query results into user-friendly text| Get row(s)               | Respond to Webhook        | Transforms query results into user-friendly format with emojis and returns via webhook.                                    |
| Respond to Webhook        | Respond to Webhook   | Sends formatted message back as HTTP response| Set Message              | —                        |                                                                                                                           |
| When clicking 'Execute workflow' | Manual Trigger  | Starts database seeding process manually      | —                        | Load Video Database       | Part of Database Initialization block: One-time setup to populate Data Table with video records. Run before using API.       |
| Load Video Database       | Code                | Provides hardcoded list of video records      | When clicking 'Execute workflow' | Loop Over Items      |                                                                                                                           |
| Loop Over Items           | Split In Batches     | Iterates over video records for insertion    | Load Video Database       | Insert row                |                                                                                                                           |
| Insert row                | Data Table Insert   | Inserts individual video record into Data Table | Loop Over Items           | Load Video Database       |                                                                                                                           |
| Main Overview             | Sticky Note         | Explains overall workflow purpose and setup  | —                        | —                        | ## 🎓 n8n Learning Hub - YouTube Educator Search\n\nSearch a curated database of n8n tutorial videos by topic using Data Tables and webhooks.\n\n## How it works\n\n**Search API** (top branch): Webhook receives search queries → normalizes keywords → queries Data Table → formats results → returns matching videos\n\n**Database Setup** (bottom branch): Manual trigger → loads 10 video records → loops through each → inserts into Data Table\n\n## Setup steps\n\n1. Create a Data Table named \"n8n_Educator_Videos\" with columns: Educator, video_title, Difficulty, YouTubeLink, Description\n2. Run the bottom branch first by clicking \"Execute workflow\" on the manual trigger node\n3. Verify 10 videos inserted into your Data Table\n4. Activate the workflow and copy the webhook Production URL\n5. Test with POST request: `{\"topic\": \"voice\"}` or `{\"topic\": \"scraping\"}`\n\nConnect your own frontend or use tools like Postman to query the API. |
| Section 1                 | Sticky Note         | Describes Search API block                     | —                        | —                        | ## Search API\n\nReceives POST requests with search topic, normalizes keywords, queries the Data Table, and returns formatted results. |
| Section 2                 | Sticky Note         | Describes response formatting                  | —                        | —                        | ## Format Response\n\nTransforms query results into user-friendly format with emojis and returns via webhook.              |
| Section 3                 | Sticky Note         | Describes Database Initialization              | —                        | —                        | ## Database Initialization\n\nOne-time setup to populate the Data Table with video records. Run this before using the search API. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Table:**  
   - Name: `n8n_Educator_Videos`  
   - Columns:  
     - `Educator` (string)  
     - `video_title` (string)  
     - `Difficulty` (string)  
     - `YouTubeLink` (string)  
     - `Description` (string)  

2. **Create Webhook Node:**  
   - Type: Webhook (HTTP Trigger)  
   - Method: POST  
   - Path: Choose a unique path (e.g., `your-unique-path`)  
   - Response Mode: Last Node  

3. **Create Code Node ("Process Search Term"):**  
   - Input: JSON with field `topic`  
   - Code:  
     ```javascript
     let topic = $json.topic || "";
     topic = topic.trim().toLowerCase();
     if (topic.includes("talk") || topic.includes("audio") || topic.includes("voice")) {
       topic = "voice";
     } else if (topic.includes("lead")) {
       topic = "lead gen";
     } else if (topic.includes("scrape") || topic.includes("data")) {
       topic = "scraping";
     }
     return [{ topic }];
     ```  

4. **Create Data Table Query Node ("Get row(s)")**  
   - Operation: Get  
   - Data Table: Select `n8n_Educator_Videos`  
   - Filters:  
     - Column: `Description`  
     - Condition: `LIKE`  
     - Value: `={{ $json["topic"] }}`  

5. **Create Set Node ("Set Message"):**  
   - Add field `Message` (string)  
   - Value:  
     ```javascript
     {{ $json["data"].map(v => `🎥 *${v["Video Title"]}* 👤 ${v["Educator"]} 🧩 Difficulty: ${v["Difficulty"]} 🔗 ${v["YouTube Link"]} 📝 ${v["Description"]}`).join("\n\n") }}
     ```  

6. **Create Respond to Webhook Node:**  
   - Default configuration to send the output from "Set Message" as HTTP response.  

7. **Connect nodes in order:**  
   - Webhook → Process Search Term → Get row(s) → Set Message → Respond to Webhook  

8. **Create Manual Trigger Node ("When clicking 'Execute workflow'")**  

9. **Create Code Node ("Load Video Database"):**  
   - JavaScript code: Define an array of 10 video objects each with fields matching Data Table columns (Educator, Video Title, Difficulty, YouTube Link, Description).  
   - Return the array mapped to `{json: videoObject}` items.  

10. **Create Split In Batches Node ("Loop Over Items"):**  
    - Default batch size (1 per iteration).  

11. **Create Data Table Insert Node ("Insert row"):**  
    - Target Data Table: `n8n_Educator_Videos`  
    - Map fields from input JSON to Data Table columns explicitly.  

12. **Connect nodes for database initialization:**  
    - Manual Trigger → Load Video Database → Loop Over Items → Insert row  

13. **Verify:**  
    - Run the Manual Trigger to seed the database.  
    - Check the Data Table contains the 10 video records.  

14. **Activate the workflow.**  
15. **Test the Webhook:**  
    - Send POST request with JSON body, e.g., `{"topic": "voice"}` or `{"topic": "scraping"}`  
    - Expect formatted list of matching videos as response.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is part of the "🎓 n8n Learning Hub" initiative to showcase practical uses of Data Tables and Webhooks for creating searchable content directories.                                                                                                                                                                         |                                                                                                                 |
| Instructions recommend running the bottom branch (database initialization) before enabling and using the search API.                                                                                                                                                                                                                     | Sticky note in "Main Overview" node.                                                                            |
| Example POST requests to test the API: `{"topic": "voice"}`, `{"topic": "scraping"}`. Use Postman or any HTTP client to test.                                                                                                                                                                                                            | Sticky note in "Main Overview" node.                                                                            |
| The workflow demonstrates simple keyword normalization to improve search relevance, which can be extended with more sophisticated NLP or AI nodes if desired.                                                                                                                                                                           | Derived from "Process Search Term" code logic.                                                                  |
| Data Table operations require proper permission and API configuration in n8n cloud or self-hosted environment with Data Table features enabled.                                                                                                                                                                                        | n8n official docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.datatable/                   |
| This workflow does not currently handle pagination or large result sets beyond what Data Tables support; consider adding limits or pagination for production use.                                                                                                                                                                        | Best practice note.                                                                                              |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a no-code integration and automation platform. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.