Automatically Update YouTube Video Descriptions with Inserted Text

https://n8nworkflows.xyz/workflows/automatically-update-youtube-video-descriptions-with-inserted-text-3080


# Automatically Update YouTube Video Descriptions with Inserted Text

### 1. Workflow Overview

This workflow automates the process of updating YouTube video descriptions by inserting a new line of text between two specified existing lines across multiple videos. It is designed primarily for YouTubers who maintain a consistent set of links or text blocks in their video descriptions and want to bulk-insert a new link or text snippet in a precise position without manual editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and setting the insertion strings (the rows before, to insert, and after).
- **1.2 Video Retrieval:** Fetching all videos from the connected YouTube channel.
- **1.3 Iterative Processing:** Looping through each video to get detailed video data.
- **1.4 Description Modification:** Parsing the existing description, inserting the new row between specified lines.
- **1.5 Update Execution:** Updating the video description on YouTube with the modified text.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initiates the workflow manually and defines the key variables that control where and what text will be inserted into the video descriptions.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set String to Insert  
  - Sticky Note (Instructional)  
  - Sticky Note1 (Configuration guidance)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow execution manually.  
    - *Configuration:* No parameters needed.  
    - *Connections:* Outputs to "Set String to Insert".  
    - *Potential Failures:* None typical; user must trigger manually.

  - **Set String to Insert**  
    - *Type:* Set  
    - *Role:* Defines three string variables controlling insertion logic:  
      - `rowBefore`: The existing line after which the new line will be inserted.  
      - `rowToInsert`: The new line/text/link to insert.  
      - `rowAfter`: The existing line before which the new line will be inserted.  
    - *Configuration:*  
      - `rowBefore` = "https://firstlink.com"  
      - `rowToInsert` = "https://mynewlinktoinsert.com"  
      - `rowAfter` = "https://secondlink.com"  
    - *Connections:* Outputs to "Get All Videos".  
    - *Edge Cases:* If these strings do not exactly match lines in descriptions, insertion will not occur.

  - **Sticky Note**  
    - *Type:* Sticky Note  
    - *Role:* Provides a high-level explanation of the workflow purpose and use case.  
    - *Content:* Describes automatic insertion of text between two specified rows in YouTube descriptions.

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Role:* Explains how to configure the insertion strings (`rowBefore`, `rowToInsert`, `rowAfter`).  
    - *Content:* Guidance on defining variables controlling insertion logic.

---

#### 2.2 Video Retrieval

- **Overview:**  
  Fetches a list of videos from the connected YouTube channel, limited to 3 videos ordered by date (most recent first).

- **Nodes Involved:**  
  - Get All Videos

- **Node Details:**

  - **Get All Videos**  
    - *Type:* YouTube node (resource: video, operation: list)  
    - *Role:* Retrieves video metadata for multiple videos.  
    - *Configuration:*  
      - Limit: 3 videos  
      - Order: Date (newest first)  
      - No filters applied (fetches all videos)  
    - *Connections:* Outputs to "Loop Over Videos".  
    - *Edge Cases:*  
      - API quota limits may affect retrieval.  
      - If no videos exist, subsequent nodes receive empty input.

---

#### 2.3 Iterative Processing

- **Overview:**  
  Processes each video individually by splitting the list into batches of one, then fetching detailed information for each video.

- **Nodes Involved:**  
  - Loop Over Videos  
  - Get Specific Video

- **Node Details:**

  - **Loop Over Videos**  
    - *Type:* SplitInBatches  
    - *Role:* Iterates over each video one at a time for sequential processing.  
    - *Configuration:*  
      - Batch size: 1 (default)  
      - Reset: false (does not reset batch index automatically)  
    - *Connections:*  
      - Input from "Get All Videos"  
      - Output batch 1 to "Get Specific Video"  
      - Output batch 2 is unused (empty)  
    - *Edge Cases:*  
      - If batch size or reset is misconfigured, may skip videos or cause infinite loops.

  - **Get Specific Video**  
    - *Type:* YouTube node (resource: video, operation: get)  
    - *Role:* Retrieves detailed metadata for a single video by videoId.  
    - *Configuration:*  
      - Video ID: dynamically set from current batch item (`{{$json.id.videoId}}`)  
    - *Connections:* Outputs to "Create New Video Description with Row Inserted".  
    - *Edge Cases:*  
      - Invalid or missing videoId causes failure.  
      - API errors or quota limits possible.

---

#### 2.4 Description Modification

- **Overview:**  
  Parses the existing video description, identifies the two specified rows, and inserts the new row between them.

- **Nodes Involved:**  
  - Create New Video Description with Row Inserted

- **Node Details:**

  - **Create New Video Description with Row Inserted**  
    - *Type:* Code (JavaScript)  
    - *Role:* Performs string manipulation on the video description.  
    - *Configuration:*  
      - Reads description from "Get Specific Video" node.  
      - Reads insertion variables from "Set String to Insert".  
      - Splits description by newline into array of rows.  
      - Finds indices of `rowBefore` and `rowAfter`.  
      - If both rows found and `rowBefore` precedes `rowAfter`, inserts `rowToInsert` between them.  
      - Joins rows back into a single string as `updatedDescription`.  
      - Returns updated description in JSON output.  
    - *Key Expressions:*  
      - Accesses `$('Get Specific Video').first().json.snippet.description`  
      - Accesses variables from `$('Set String to Insert').first().json`  
    - *Connections:* Outputs to "Update Video Description".  
    - *Edge Cases:*  
      - If either `rowBefore` or `rowAfter` is not found, no insertion occurs.  
      - If `rowBefore` comes after `rowAfter`, no insertion occurs.  
      - Description with different formatting or extra whitespace may cause index lookup failures.  
      - Large descriptions may impact performance.  
      - Errors in JavaScript code could halt execution.

---

#### 2.5 Update Execution

- **Overview:**  
  Updates the video description on YouTube with the newly constructed description.

- **Nodes Involved:**  
  - Update Video Description

- **Node Details:**

  - **Update Video Description**  
    - *Type:* YouTube node (resource: video, operation: update)  
    - *Role:* Sends the updated description and other metadata back to YouTube to update the video.  
    - *Configuration:*  
      - Video ID: from "Get Specific Video" (`{{$json.id}}`)  
      - Title: from "Get Specific Video" (`{{$json.snippet.title}}`)  
      - Category ID: from "Get Specific Video" (`{{$json.snippet.categoryId}}`)  
      - Region Code: fixed to "US"  
      - Tags: from "Get Specific Video" (`{{$json.snippet.tags.join()}}`)  
      - Description: from "Create New Video Description with Row Inserted" (`{{$json.updatedDescription}}`)  
    - *Connections:* Outputs back to "Loop Over Videos" (to continue batch processing).  
    - *Edge Cases:*  
      - API quota or permission errors may block updates.  
      - If tags are missing or empty, joining may cause issues (should be handled gracefully).  
      - Updating with invalid data may cause YouTube API errors.  
      - Region code hardcoded to "US" may not be appropriate for all users.

---

### 3. Summary Table

| Node Name                          | Node Type          | Functional Role                              | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                                         |
|-----------------------------------|--------------------|----------------------------------------------|-------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger     | Starts workflow manually                      |                         | Set String to Insert           |                                                                                                                     |
| Set String to Insert               | Set                | Defines insertion strings (rowBefore, rowToInsert, rowAfter) | When clicking ‘Test workflow’ | Get All Videos                 |                                                                                                                     |
| Sticky Note                       | Sticky Note        | Explains workflow purpose and use case       |                         |                               | ## Insert Text into YouTube Video Descriptions\nAutomatically insert a row of text between two specified rows in all your YouTube video descriptions. This workflow is ideal for YouTubers who need to update multiple video descriptions at once. Easily add a new link or text between existing lines, ensuring consistency across all your video descriptions without manual edits. |
| Sticky Note1                      | Sticky Note        | Explains how to configure insertion strings  |                         |                               | ## Configure text string to insert 👆 \nDefine the text string (row) that will be added to your YouTube video descriptions.\n\n### Variables\n- **rowBefore** → The new row will be inserted *after* this line.\n- **rowToInsert** -→ The text or link you want to add.\n- **rowAfter**→ The new row will be inserted *before* this line.\n\n |
| Get All Videos                   | YouTube            | Retrieves list of videos from channel        | Set String to Insert     | Loop Over Videos               |                                                                                                                     |
| Loop Over Videos                | SplitInBatches     | Iterates over videos one by one               | Get All Videos           | Get Specific Video             |                                                                                                                     |
| Get Specific Video              | YouTube            | Retrieves detailed info for one video         | Loop Over Videos         | Create New Video Description with Row Inserted |                                                                                                                     |
| Create New Video Description with Row Inserted | Code (JavaScript) | Inserts new row between specified lines in description | Get Specific Video       | Update Video Description       |                                                                                                                     |
| Update Video Description        | YouTube            | Updates video description on YouTube          | Create New Video Description with Row Inserted | Loop Over Videos               |                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Create a Set node**  
   - Name: `Set String to Insert`  
   - Purpose: Define three string variables for insertion logic:  
     - `rowBefore` = `"https://firstlink.com"`  
     - `rowToInsert` = `"https://mynewlinktoinsert.com"`  
     - `rowAfter` = `"https://secondlink.com"`  
   - Connect output of Manual Trigger to this node.

3. **Create a YouTube node**  
   - Name: `Get All Videos`  
   - Resource: `video`  
   - Operation: `list`  
   - Parameters:  
     - Limit: 3  
     - Order: `date` (newest first)  
   - Connect output of `Set String to Insert` to this node.  
   - Credentials: Connect your YouTube account with appropriate scopes (read access to videos).

4. **Create a SplitInBatches node**  
   - Name: `Loop Over Videos`  
   - Purpose: Process videos one at a time.  
   - Parameters: Default batch size (1), Reset: false.  
   - Connect output of `Get All Videos` to this node.

5. **Create a YouTube node**  
   - Name: `Get Specific Video`  
   - Resource: `video`  
   - Operation: `get`  
   - Parameters:  
     - Video ID: `={{ $json.id.videoId }}` (dynamic from batch item)  
   - Connect output batch 1 of `Loop Over Videos` to this node.

6. **Create a Code node**  
   - Name: `Create New Video Description with Row Inserted`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const description = $('Get Specific Video').first().json.snippet.description;
     const variables = $('Set String to Insert').first().json;
     const rowBefore = variables.rowBefore;
     const rowAfter = variables.rowAfter;
     const rowToInsert = variables.rowToInsert;
     const rows = description.split("\n");
     const indexBefore = rows.findIndex(row => row.trim() === rowBefore);
     const indexAfter = rows.findIndex(row => row.trim() === rowAfter);
     if (indexBefore !== -1 && indexAfter !== -1 && indexBefore < indexAfter) {
       rows.splice(indexBefore + 1, 0, rowToInsert);
     }
     const updatedDescription = rows.join("\n");
     return [{ json: { updatedDescription } }];
     ```
   - Connect output of `Get Specific Video` to this node.

7. **Create a YouTube node**  
   - Name: `Update Video Description`  
   - Resource: `video`  
   - Operation: `update`  
   - Parameters:  
     - Video ID: `={{ $('Get Specific Video').item.json.id }}`  
     - Title: `={{ $('Get Specific Video').item.json.snippet.title }}`  
     - Category ID: `={{ $('Get Specific Video').item.json.snippet.categoryId }}`  
     - Region Code: `"US"` (hardcoded)  
     - Tags: `={{ $('Get Specific Video').item.json.snippet.tags.join() }}`  
     - Description: `={{ $json.updatedDescription }}` (from Code node)  
   - Connect output of Code node to this node.

8. **Connect output of `Update Video Description` back to `Loop Over Videos`**  
   - This allows the batch processing to continue for all videos.

9. **Credential Setup:**  
   - Ensure your YouTube OAuth2 credentials are configured with permissions to read and update videos.  
   - No other credentials are required.

10. **Optional:** Add Sticky Notes for documentation and configuration guidance as per your preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is ideal for YouTubers who want to bulk-update video descriptions by inserting new links/text. | Workflow description and use case summary.                                                     |
| Variables `rowBefore`, `rowToInsert`, and `rowAfter` must exactly match lines in descriptions for insertion. | Configuration note to avoid insertion failures.                                                |
| YouTube API quota limits and permissions can affect workflow execution.                                       | General API usage consideration.                                                              |
| To customize insertion logic or filter videos, modify the "Set String to Insert" node or "Get All Videos" node. | Customization guidance.                                                                         |
| Hardcoded region code "US" in update node may need adjustment for non-US channels.                            | Localization consideration.                                                                    |

---

This completes the comprehensive reference document for the "Automatically Update YouTube Video Descriptions with Inserted Text" workflow. It enables understanding, reproduction, and modification with attention to potential edge cases and integration points.