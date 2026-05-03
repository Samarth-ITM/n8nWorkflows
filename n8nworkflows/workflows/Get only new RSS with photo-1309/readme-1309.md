Get only new RSS with photo

https://n8nworkflows.xyz/workflows/get-only-new-rss-with-photo-1309


# Get only new RSS with photo

### 1. Workflow Overview

This workflow is designed to fetch only new RSS feed items that contain photos, effectively filtering out previously processed entries and extracting the primary image from each new post. It targets users who want to automate the retrieval of fresh RSS content with images for further use, such as posting to messaging services (e.g., Telegram).  
The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Regularly triggers the workflow every 5 minutes.  
- **1.2 RSS Feed Reading:** Retrieves the latest RSS feed entries from a specified URL.  
- **1.3 Data Filtering and Structuring:** Selects and formats key RSS data fields for further processing.  
- **1.4 New RSS Detection:** Compares current feed items against stored IDs to isolate only new entries.  
- **1.5 Image Extraction:** Parses the content of new RSS entries to extract the first image URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically on a time schedule, ensuring periodic execution without manual intervention.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  
  - **Cron**  
    - Type: `Cron` node (Trigger)  
    - Configuration: Set to trigger every 5 minutes (using “every X minutes” mode with value 5)  
    - Key Expressions/Variables: None  
    - Input Connections: None (entry point)  
    - Output Connections: To `RSS Feed Read` node  
    - Version Requirements: Compatible with n8n version supporting Cron node (standard)  
    - Edge Cases/Potential Failures: Misconfiguration could cause too frequent or infrequent triggers; system time errors could affect schedule  

#### 1.2 RSS Feed Reading

- **Overview:**  
  Reads the full RSS feed from The Verge’s feed URL, fetching the latest posts at each trigger.

- **Nodes Involved:**  
  - RSS Feed Read

- **Node Details:**  
  - **RSS Feed Read**  
    - Type: `RSS Feed Read` node  
    - Configuration: URL set to `http://www.theverge.com/rss/full.xml`  
    - Execute Once: True (ensures single execution per trigger)  
    - Key Expressions/Variables: None beyond URL  
    - Input Connections: From `Cron`  
    - Output Connections: To `Filter RSS Data`  
    - Version Requirements: None special; standard RSS support  
    - Edge Cases/Potential Failures: Feed URL unreachable, network issues, malformed feed XML  

#### 1.3 Data Filtering and Structuring

- **Overview:**  
  Extracts and restructures relevant RSS item fields (title, subtitle, author, URL, publication date, and content) into a simplified JSON object for further processing.

- **Nodes Involved:**  
  - Filter RSS Data

- **Node Details:**  
  - **Filter RSS Data**  
    - Type: `Set` node  
    - Configuration: Uses expressions to map RSS fields to custom names:
      - Title ← RSS item’s `title`  
      - Subtitle ← `contentSnippet`  
      - Author ← `creator`  
      - URL ← RSS item’s `link`  
      - Date ← RSS item’s `pubDate`  
      - content ← raw `content` HTML  
    - Options: Keeps only these set fields (removes all others)  
    - Key Expressions/Variables: Expressions like `{{$node["RSS Feed Read"].json["title"]}}` and `{{$json["contentSnippet"]}}`  
    - Input Connections: From `RSS Feed Read`  
    - Output Connections: To `Only get new RSS1`  
    - Version Requirements: Expression syntax compatible with n8n versions supporting mustache expressions  
    - Edge Cases/Potential Failures: Missing or null RSS fields could lead to empty values; expression errors if node names change  

#### 1.4 New RSS Detection

- **Overview:**  
  Filters out any RSS entries that were already processed in previous workflow runs by comparing publication dates stored in workflow static data.

- **Nodes Involved:**  
  - Only get new RSS1

- **Node Details:**  
  - **Only get new RSS1**  
    - Type: `Function` node  
    - Configuration: JavaScript code accessing workflow static data to track previously seen publication dates (stored under `oldRSSIds`):  
      - On first run, stores all current dates and returns all items.  
      - On subsequent runs, filters items to only those with dates not previously stored.  
      - Updates stored dates with new ones appended.  
    - Key Expressions/Variables: Accesses static data via `getWorkflowStaticData('global')`  
    - Input Connections: From `Filter RSS Data`  
    - Output Connections: To `Extract Image1`  
    - Version Requirements: Requires n8n version supporting workflow static data (standard in recent versions)  
    - Edge Cases/Potential Failures:  
      - If date formats vary or are inconsistent, filtering might fail.  
      - Static data persistence limits could impact long-term operation.  
      - Multiple RSS items with identical dates might cause duplicates or misses.  

#### 1.5 Image Extraction

- **Overview:**  
  Parses the HTML content in the filtered new RSS items to extract the `src` attribute of the first image found.

- **Nodes Involved:**  
  - Extract Image1

- **Node Details:**  
  - **Extract Image1**  
    - Type: `HTML Extract` node  
    - Configuration:  
      - Input data property: `content` (from previous node’s JSON)  
      - CSS selector: `img` (selects first image element)  
      - Attribute to extract: `src` attribute (returns image URL)  
    - Key Expressions/Variables: Uses `=content` to access HTML content  
    - Input Connections: From `Only get new RSS1`  
    - Output Connections: None (end of current workflow steps)  
    - Version Requirements: Compatible with latest n8n HTML Extract node features  
    - Edge Cases/Potential Failures:  
      - If no image exists in content, output will be empty or null.  
      - HTML content malformed or empty could cause extraction failure.  

---

### 3. Summary Table

| Node Name       | Node Type           | Functional Role            | Input Node(s)   | Output Node(s)  | Sticky Note                                                                                     |
|-----------------|---------------------|----------------------------|-----------------|-----------------|------------------------------------------------------------------------------------------------|
| Cron            | Cron Trigger        | Scheduled trigger every 5 min | None            | RSS Feed Read   |                                                                                                |
| RSS Feed Read   | RSS Feed Reader      | Fetches latest RSS posts    | Cron            | Filter RSS Data |                                                                                                |
| Filter RSS Data | Set                  | Selects and formats key RSS fields | RSS Feed Read   | Only get new RSS1 |                                                                                                |
| Only get new RSS1 | Function            | Filters only new RSS items based on date | Filter RSS Data | Extract Image1  |                                                                                                |
| Extract Image1  | HTML Extract         | Extracts first image URL from content | Only get new RSS1 | None           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it “Get only new RSS with Photo.”

2. **Add a Cron node:**  
   - Set trigger mode to “Every X minutes”  
   - Value: 5  
   - Position: Top-left or as preferred  
   - No credentials needed.

3. **Add an RSS Feed Read node:**  
   - Set URL to `http://www.theverge.com/rss/full.xml`  
   - Ensure “Execute Once” is enabled to avoid multiple fetches per trigger  
   - Connect Cron node’s output to this node’s input.

4. **Add a Set node (Filter RSS Data):**  
   - Configure with the following fields (all as strings, using expressions):  
     - Title: `{{$node["RSS Feed Read"].json["title"]}}`  
     - Subtitle: `{{$json["contentSnippet"]}}`  
     - Author: `{{$json["creator"]}}`  
     - URL: `{{$node["RSS Feed Read"].json["link"]}}`  
     - Date: `{{$node["RSS Feed Read"].json["pubDate"]}}`  
     - content: `{{$json["content"]}}`  
   - Enable “Keep Only Set” option to remove all other data  
   - Connect RSS Feed Read node output to this Set node.

5. **Add a Function node (Only get new RSS1):**  
   - Paste the following JavaScript code:
   ```javascript
   const staticData = getWorkflowStaticData('global');
   const newRSSIds = items.map(item => item.json["Date"]);
   const oldRSSIds = staticData.oldRSSIds;

   if (!oldRSSIds) {
     staticData.oldRSSIds = newRSSIds;
     return items;
   }

   const actualNewRSSIds = newRSSIds.filter((id) => !oldRSSIds.includes(id));
   const actualNewRSS = items.filter((data) => actualNewRSSIds.includes(data.json['Date']));
   staticData.oldRSSIds = [...actualNewRSSIds, ...oldRSSIds];

   return actualNewRSS;
   ```
   - Connect Set node output to this Function node.

6. **Add an HTML Extract node (Extract Image1):**  
   - Set “Data Property Name” to `=content` (this tells the node to extract from the `content` field)  
   - Add extraction value:  
     - Key: `image`  
     - CSS Selector: `img`  
     - Return Value: `attribute`  
     - Attribute: `src`  
   - Connect Function node output to this HTML Extract node.

7. **Final Step (Optional):**  
   Add any further nodes to send or use the extracted new RSS items with images, e.g., Telegram node configured with appropriate credentials to post the extracted content.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow result is only visible upon manual or scheduled execution, not in idle state.          | Workflow behavior note                                                                                 |
| Based on n8n community examples: “Latest RSS Feed -> Rocket.Chat” and “RSS to Twitter with Image.”   | https://community.n8n.io/t/latest-rss-feed-rocket-chat/3770 & https://community.n8n.io/t/rss-to-twitter-with-image/4282 |
| To extend functionality, add service nodes like Telegram after image extraction to push updates.   | Workflow description                                                                                   |

---

This document fully describes the “Get only new RSS with Photo” workflow for both human operators and automation agents, enabling understanding, reproduction, and troubleshooting.