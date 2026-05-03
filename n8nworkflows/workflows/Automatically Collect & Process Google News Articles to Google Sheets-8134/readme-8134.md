Automatically Collect & Process Google News Articles to Google Sheets

https://n8nworkflows.xyz/workflows/automatically-collect---process-google-news-articles-to-google-sheets-8134


# Automatically Collect & Process Google News Articles to Google Sheets

### 1. Workflow Overview

This workflow automates the collection, deduplication, cleaning, and saving of Google News articles related to specific AI topics into a Google Sheets document on a weekly schedule. It targets use cases such as media monitoring, content aggregation, and news analysis by fetching RSS feeds, processing entries, and appending unique, cleaned article data to a spreadsheet.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Scheduled trigger and RSS feed reading for two AI-related topics.
- **1.2 Data Merging and Cleaning:** Consolidation of RSS feed items, URL extraction and normalization, filtering for completeness, and removal of duplicates.
- **1.3 Output Storage:** Appending the cleaned, unique article data into a Google Sheets document.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  This block triggers the workflow every Monday at 9:00 am and reads two Google News RSS feeds for different AI-related topics in German for Germany.

- **Nodes Involved:**  
  - Trigger: Monday 09:00  
  - RSS Read — Google News (TOPIC_1)  
  - RSS Read — Google News (TOPIC_2)  
  - Merge Feeds (Append)

- **Node Details:**

  1. **Trigger: Monday 09:00**  
     - Type: Cron Trigger  
     - Role: Starts the workflow weekly at 09:00 on Mondays.  
     - Configuration: Scheduled to trigger every week at 9:00 AM local time.  
     - Input: None (trigger node)  
     - Output: Starts RSS feed reading nodes in parallel.  
     - Edge Cases: Cron misconfiguration, timezone discrepancies.  
     - Version: n8n v1+ compatible.

  2. **RSS Read — Google News (TOPIC_1)**  
     - Type: RSS Feed Read  
     - Role: Reads RSS feed for the search query "AI" localized for Germany (German language).  
     - Configuration: URL set to `https://news.google.com/rss/search?q=AI&hl=de&gl=DE&ceid=DE:de`.  
     - Input: Trigger node output.  
     - Output: List of RSS feed items for topic 1.  
     - Edge Cases: RSS feed downtime, network errors, empty feeds.  
     - Version: Standard RSS node.

  3. **RSS Read — Google News (TOPIC_2)**  
     - Type: RSS Feed Read  
     - Role: Reads RSS feed for the search query "AIAutomation" localized for Germany (German language).  
     - Configuration: URL set to `https://news.google.com/rss/search?q=AIAutomation&hl=de&gl=DE&ceid=DE:de`.  
     - Input: Trigger node output.  
     - Output: List of RSS feed items for topic 2.  
     - Edge Cases: Similar to Topic 1 node.

  4. **Merge Feeds (Append)**  
     - Type: Merge (Append mode)  
     - Role: Combines the two RSS feed outputs into a single item list for downstream processing.  
     - Configuration: Default append (no special options).  
     - Input: Parallel inputs from both RSS Read nodes.  
     - Output: Combined list of feed items.  
     - Edge Cases: Handling empty inputs from either feed; ensures no data loss.  
     - Version: v2+ node for merging.

---

#### Block 1.2: Data Merging and Cleaning

- **Overview:**  
  This block normalizes URLs and selects key fields, filters to keep only complete entries, and removes duplicate articles based on URLs to ensure data quality.

- **Nodes Involved:**  
  - Set (Clean URL + Fields)  
  - Filter (only complete Items)  
  - Item Lists: Unique by URL

- **Node Details:**

  1. **Set (Clean URL + Fields)**  
     - Type: Set  
     - Role: Extracts and normalizes the article URL, selects relevant fields (title, summary, publication date), and cleans data for consistency.  
     - Configuration:  
       - The `url` field is extracted by parsing the original `link` property for a `url` query parameter; if missing, falls back to `link` or `guid`.  
       - Uses JavaScript expressions to decode URL parameters and fallback logic.  
       - Sets `title`, `summary` (from `contentSnippet` or `summary`), and `pubDate` (from multiple date fields) with fallbacks to empty strings if missing.  
       - Keeps only these fields (drops others).  
     - Input: Combined feed items from Merge Feeds (Append).  
     - Output: Cleaned and standardized article objects.  
     - Edge Cases: Links without query parameters, missing fields, malformed URLs causing expression failures.  
     - Version: v2 node with expression support.

  2. **Filter (only complete Items)**  
     - Type: Filter  
     - Role: Removes items lacking a non-empty `title` or `url` to ensure only valid, complete articles proceed.  
     - Configuration: Filters with an AND condition requiring both `title` and `url` fields to be non-empty strings.  
     - Input: Output from Set node.  
     - Output: Filtered list of complete article items.  
     - Edge Cases: Items with missing or empty fields filtered out; no output if all items invalid.

  3. **Item Lists: Unique by URL**  
     - Type: Item Lists (Remove Duplicates)  
     - Role: Removes duplicate articles by comparing the `url` field to ensure only unique news entries are saved.  
     - Configuration: Removes duplicates based on the `url` property.  
     - Input: Filtered articles from Filter node.  
     - Output: Unique articles list.  
     - Edge Cases: Case sensitivity in URLs, slight URL variations may cause duplicates to persist.

---

#### Block 1.3: Output Storage

- **Overview:**  
  This block appends the final list of unique, cleaned articles to a Google Sheets document for archival or further analysis.

- **Nodes Involved:**  
  - Append new Links (Google Sheets node)

- **Node Details:**

  1. **Append new Links**  
     - Type: Google Sheets  
     - Role: Appends or updates rows with new article data in the specified Google Sheets document and sheet.  
     - Configuration:  
       - Operation: `appendOrUpdate` — adds new rows or updates existing ones based on key fields.  
       - `documentId` and `sheetName` are set via list mode parameters (likely referencing external input or environment variables).  
       - Uses Google Sheets OAuth2 credentials.  
     - Input: Unique article list from Item Lists: Unique by URL node.  
     - Output: Appended rows in Google Sheets (no further nodes).  
     - Edge Cases: Authentication failure, quota limits, sheet not found, permission issues, malformed data.  
     - Version: v4 Google Sheets node.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                         | Input Node(s)                      | Output Node(s)                  | Sticky Note                           |
|-------------------------------|---------------------|--------------------------------------|----------------------------------|--------------------------------|-------------------------------------|
| Trigger: Monday 09:00          | Cron Trigger        | Weekly schedule trigger               | None                             | RSS Read — Google News (TOPIC_2), RSS Read — Google News (TOPIC_1) |                                     |
| RSS Read — Google News (TOPIC_1) | RSS Feed Read       | Fetches RSS feed for "AI" topic       | Trigger: Monday 09:00             | Merge Feeds (Append)            |                                     |
| RSS Read — Google News (TOPIC_2) | RSS Feed Read       | Fetches RSS feed for "AIAutomation" topic | Trigger: Monday 09:00             | Merge Feeds (Append)            |                                     |
| Merge Feeds (Append)           | Merge               | Combines RSS feed outputs             | RSS Read — TOPIC_1, RSS Read — TOPIC_2 | Set (Clean URL + Fields)        |                                     |
| Set (Clean URL + Fields)       | Set                 | Normalizes URLs and selects fields    | Merge Feeds (Append)              | Filter (only complete Items)    |                                     |
| Filter (only complete Items)   | Filter              | Filters out incomplete articles       | Set (Clean URL + Fields)          | Item Lists: Unique by URL       |                                     |
| Item Lists: Unique by URL      | Item Lists          | Removes duplicate articles by URL     | Filter (only complete Items)      | Append new Links                |                                     |
| Append new Links               | Google Sheets       | Appends unique articles to Google Sheets | Item Lists: Unique by URL          | None                           |                                     |
| Sticky: 🧼 Clean & Neu         | Sticky Note         | Section label: Gather Information     | None                             | None                           | ## Gather Information               |
| Sticky: 🧼 Clean & Neu2        | Sticky Note         | Section label: Clean & New             | None                             | None                           | ## Clean & New                     |
| Sticky: 🧼 Clean & Neu1        | Sticky Note         | Section label: Save in Sheets          | None                             | None                           | ## Save in Sheets                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger node:**  
   - Type: Cron Trigger  
   - Name: "Trigger: Monday 09:00"  
   - Set to trigger weekly every Monday at 09:00 local time.  

2. **Create RSS Feed Read nodes:**  
   - Node 1:  
     - Type: RSS Feed Read  
     - Name: "RSS Read — Google News (TOPIC_1)"  
     - URL: `https://news.google.com/rss/search?q=AI&hl=de&gl=DE&ceid=DE:de`  
   - Node 2:  
     - Type: RSS Feed Read  
     - Name: "RSS Read — Google News (TOPIC_2)"  
     - URL: `https://news.google.com/rss/search?q=AIAutomation&hl=de&gl=DE&ceid=DE:de`  
   - Connect the output of the Cron Trigger node to both RSS Feed Read nodes (parallel connections).  

3. **Create Merge node:**  
   - Type: Merge  
   - Name: "Merge Feeds (Append)"  
   - Mode: Append (default)  
   - Connect RSS Read — Google News (TOPIC_1) output to input 1 of Merge node.  
   - Connect RSS Read — Google News (TOPIC_2) output to input 2 of Merge node.  

4. **Create Set node:**  
   - Type: Set  
   - Name: "Set (Clean URL + Fields)"  
   - Configure to extract and clean the following fields:  
     - `url`: Extract from `link` query parameter `url` if present; else fallback to `link` or `guid`; decode URL component.  
     - `title`: Use `title` field or empty string.  
     - `summary`: Use `contentSnippet` or `summary` or empty string.  
     - `pubDate`: Use `pubDate`, `isoDate`, or `date` or empty string.  
   - Enable "Keep Only Set" option to discard other fields.  
   - Connect Merge node output to Set node input.  

5. **Create Filter node:**  
   - Type: Filter  
   - Name: "Filter (only complete Items)"  
   - Add AND condition:  
     - `title` is not empty  
     - `url` is not empty  
   - Connect Set node output to Filter node input.  

6. **Create Item Lists node:**  
   - Type: Item Lists  
   - Name: "Item Lists: Unique by URL"  
   - Operation: Remove duplicates based on `url` field.  
   - Connect Filter node output to Item Lists node input.  

7. **Create Google Sheets node:**  
   - Type: Google Sheets  
   - Name: "Append new Links"  
   - Operation: Append or Update rows  
   - Set `documentId` and `sheetName` to the target Google Sheets document ID and sheet name respectively (use list mode or fixed values).  
   - Authenticate with Google Sheets OAuth2 credentials (configure a credential with necessary permissions).  
   - Connect Item Lists node output to Google Sheets node input.  

8. **Add Sticky Notes for documentation (optional):**  
   - Create three sticky notes positioned to visually group the blocks:  
     - "## Gather Information" near the input nodes.  
     - "## Clean & New" near the Set, Filter, and Item Lists nodes.  
     - "## Save in Sheets" near the Google Sheets node.  

---

### 5. General Notes & Resources

| Note Content                                                    | Context or Link                          |
|----------------------------------------------------------------|-----------------------------------------|
| This workflow is designed for German-language Google News RSS feeds targeting AI topics. Adjust RSS URLs for other languages or regions as needed. | Localization detail in RSS URLs          |
| Google Sheets OAuth2 credentials must have Edit access to the target document to append data successfully. | Google Sheets OAuth2 credential setup   |
| URL extraction logic assumes article URLs are sometimes embedded in query parameters; adjust regex if feed structure changes. | Expression detail in "Set (Clean URL + Fields)" node |
| Scheduling is weekly on Mondays at 09:00; adjust cron node for different frequencies or timezones. | Cron trigger configuration               |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.