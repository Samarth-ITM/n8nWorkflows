📌 Turn your LinkedIn insights into content ideas with Airtable and Real-Time Linkedin Scraper

https://n8nworkflows.xyz/workflows/---turn-your-linkedin-insights-into-content-ideas-with-airtable-and-real-time-linkedin-scraper-3520


# 📌 Turn your LinkedIn insights into content ideas with Airtable and Real-Time Linkedin Scraper

### 1. Workflow Overview

This workflow automates the transformation of your recent LinkedIn post reactions into structured content ideas stored in Airtable. It is designed to help content creators, marketers, and personal brand builders capture inspiration from LinkedIn activity and organize it for brainstorming or publication planning.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Periodically initiates the workflow.
- **1.2 LinkedIn Data Retrieval:** Fetches recent liked posts from LinkedIn using a public API via RapidAPI.
- **1.3 Data Splitting:** Breaks down the batch of liked posts into individual items for processing.
- **1.4 Filtering:** Selects only posts marked as “insightful” and posted within the last 7 days.
- **1.5 Content Formatting:** Converts filtered posts into a structured format suitable for Airtable.
- **1.6 Airtable Preparation:** Prepares and maps the formatted data fields to Airtable columns.
- **1.7 Data Storage:** Saves the structured content ideas into a specified Airtable base and table.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow execution on a defined schedule to automate periodic data fetching.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Configuration:** Default interval set (user-configurable) to run the workflow regularly.  
  - **Expressions/Variables:** None.  
  - **Input:** None (trigger node).  
  - **Output:** Triggers the next node, "Fetch LinkedIn Likes".  
  - **Version Requirements:** n8n version supporting scheduleTrigger v1.2 or higher.  
  - **Edge Cases:** Misconfigured schedule may cause no runs or excessive runs.  
  - **Sub-workflow:** None.

#### 2.2 LinkedIn Data Retrieval

- **Overview:**  
  Fetches the latest liked posts of a LinkedIn user via the RapidAPI endpoint `linkedin-api8`.

- **Nodes Involved:**  
  - Fetch LinkedIn Likes

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - Method: GET (default for HTTP Request node).  
    - URL: `https://linkedin-api8.p.rapidapi.com/get-profile-likes`  
    - Query Parameters:  
      - `username`: User’s LinkedIn username (must be replaced by user).  
      - `start`: Pagination start index, set to "0" to fetch from the beginning.  
    - Headers:  
      - `x-rapidapi-host`: `linkedin-api8.p.rapidapi.com`  
      - `x-rapidapi-key`: User’s RapidAPI key (must be replaced by user).  
  - **Expressions:** Uses expressions to insert username and API key dynamically.  
  - **Input:** Trigger from Schedule Trigger.  
  - **Output:** JSON response containing liked posts data.  
  - **Version Requirements:** HTTP Request node v4.2 or newer recommended for query and header parameter support.  
  - **Edge Cases:**  
    - API key invalid or quota exceeded → HTTP 401/429 errors.  
    - Username invalid → empty or error response.  
    - Network timeouts or API downtime.  
  - **Sub-workflow:** None.

#### 2.3 Data Splitting

- **Overview:**  
  Splits the array of liked posts into individual items for granular processing.

- **Nodes Involved:**  
  - Split Liked Posts

- **Node Details:**  
  - **Type:** Split Out  
  - **Configuration:**  
    - Field to split out: `data.items` (the array containing liked posts).  
  - **Input:** Output from "Fetch LinkedIn Likes".  
  - **Output:** Individual liked post JSON objects.  
  - **Version Requirements:** Split Out node v1.  
  - **Edge Cases:**  
    - Empty `data.items` array → no outputs, workflow ends gracefully.  
  - **Sub-workflow:** None.

#### 2.4 Filtering Insightful & Recent Posts

- **Overview:**  
  Filters posts to keep only those marked with the reaction "insightful" and posted within the last 7 days.

- **Nodes Involved:**  
  - Filter Insightful & Recent

- **Node Details:**  
  - **Type:** Filter  
  - **Configuration:**  
    - Conditions combined with AND:  
      1. `$json.action` contains `"insightful"` (case-sensitive).  
      2. `postedDate` timestamp is greater than current time minus 7 days (in milliseconds).  
  - **Expressions:**  
    - Uses JavaScript date expressions to compare timestamps dynamically.  
  - **Input:** Individual liked post from "Split Liked Posts".  
  - **Output:** Posts passing both conditions forwarded to next node.  
  - **Version Requirements:** Filter node v2.2 or higher for advanced expression support.  
  - **Edge Cases:**  
    - Posts without `action` or `postedDate` fields → may fail or be excluded.  
    - Timezone differences handled by JavaScript Date object.  
  - **Sub-workflow:** None.

#### 2.5 Content Formatting

- **Overview:**  
  Formats each filtered post into a structured content idea with title, description, and source URL.

- **Nodes Involved:**  
  - Format Content Idea

- **Node Details:**  
  - **Type:** Set  
  - **Configuration:**  
    - Assigns new fields:  
      - `Title`: `"I just liked a linkedin post of {{ $json.author.username }}"`  
      - `description`: Post text content from `$json.text`  
      - `source`: URL of the post from `$json.postUrl`  
  - **Expressions:** Uses mustache-style templating to dynamically build fields.  
  - **Input:** Filtered post JSON.  
  - **Output:** JSON with new structured fields.  
  - **Version Requirements:** Set node v3.4 or higher for advanced expressions.  
  - **Edge Cases:**  
    - Missing author username or post text → incomplete title or description.  
  - **Sub-workflow:** None.

#### 2.6 Airtable Preparation

- **Overview:**  
  Prepares the formatted content idea data for Airtable by splitting out the relevant fields.

- **Nodes Involved:**  
  - Prepare for Airtable

- **Node Details:**  
  - **Type:** Split Out  
  - **Configuration:**  
    - Fields to split out: `Title, description, source`  
  - **Input:** Output from "Format Content Idea".  
  - **Output:** Individual fields ready for Airtable mapping.  
  - **Version Requirements:** Split Out node v1.  
  - **Edge Cases:**  
    - Missing any of the three fields may cause incomplete Airtable records.  
  - **Sub-workflow:** None.

#### 2.7 Data Storage in Airtable

- **Overview:**  
  Creates new records in an Airtable base/table with the content ideas.

- **Nodes Involved:**  
  - Save to Airtable

- **Node Details:**  
  - **Type:** Airtable  
  - **Configuration:**  
    - Base: `Content Hub` (ID: `appgNpFtbtaGHM4g0`)  
    - Table: `Ideas` (ID: `tblwBVudDpOMkUGKL`)  
    - Columns mapped:  
      - `Type`: fixed value `"Linkedin"`  
      - `Title`: from `$json.Title`  
      - `Source`: from `$json.source`  
      - `Status`: set to `false` (boolean)  
      - `Description`: from `$json.description`  
    - Mapping mode: Define below with explicit field mapping.  
  - **Credentials:** Airtable Personal Access Token configured in n8n credentials.  
  - **Input:** Prepared fields from "Prepare for Airtable".  
  - **Output:** Confirmation of record creation.  
  - **Version Requirements:** Airtable node v2.1 or higher recommended.  
  - **Edge Cases:**  
    - Airtable API limits or authentication failure.  
    - Missing required fields in Airtable base schema.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                    | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                           |
|---------------------|--------------------|----------------------------------|-----------------------|---------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger   | Initiates workflow on schedule   | None                  | Fetch LinkedIn Likes       |                                                                                                     |
| Fetch LinkedIn Likes | HTTP Request       | Fetches liked posts from LinkedIn| Schedule Trigger      | Split Liked Posts          |                                                                                                     |
| Split Liked Posts    | Split Out          | Splits batch into individual posts| Fetch LinkedIn Likes  | Filter Insightful & Recent |                                                                                                     |
| Filter Insightful & Recent | Filter         | Filters posts by reaction and date| Split Liked Posts     | Format Content Idea        |                                                                                                     |
| Format Content Idea  | Set                | Formats post data into content idea| Filter Insightful & Recent | Prepare for Airtable    |                                                                                                     |
| Prepare for Airtable | Split Out          | Prepares fields for Airtable     | Format Content Idea    | Save to Airtable           |                                                                                                     |
| Save to Airtable     | Airtable           | Saves content ideas to Airtable  | Prepare for Airtable   | None                      |                                                                                                     |
| Sticky Note         | Sticky Note        | Describes workflow purpose       | None                  | None                      | ## 📝 Description\nAutomatically turn your insightful LinkedIn post reactions into structured content ideas saved in Airtable. This workflow fetches your recent *"insightful"* likes, filters for posts from the last 7 days, extracts relevant content, and logs it into Airtable for future content inspiration. |
| Sticky Note1        | Sticky Note        | Explains workflow functionality  | None                  | None                      | ## ⚙️ What It Does\n- **Fetches** recent liked posts from LinkedIn using RapidAPI.\n- **Filters** only *insightful* reactions from the past 7 days.\n- **Structures** each post into a title, description, and source URL.\n- **Stores** the content in a custom Airtable base. |
| Sticky Note2        | Sticky Note        | Setup instructions for users     | None                  | None                      | ## 🧰 Setup Instructions\n1. Clone this template into your n8n instance.\n2. Open the `Fetch LinkedIn Likes` node and enter:\n   - Your LinkedIn username.\n   - Your RapidAPI key in the headers.\n3. Open the `Save to Airtable` node and:\n   - Connect your Airtable account.\n   - Link the correct base (`Content Hub`) and table (`Ideas`).\n4. Set your desired schedule in the `Trigger` node.\n5. Activate the workflow and you're done! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Linkedin to Airtable").

2. **Add a Schedule Trigger node:**  
   - Node Type: Schedule Trigger  
   - Set the desired interval (e.g., daily, hourly) to automate workflow runs.

3. **Add an HTTP Request node:**  
   - Name: "Fetch LinkedIn Likes"  
   - Method: GET  
   - URL: `https://linkedin-api8.p.rapidapi.com/get-profile-likes`  
   - Query Parameters:  
     - `username`: Set as your LinkedIn username (replace placeholder).  
     - `start`: Set to "0".  
   - Headers:  
     - `x-rapidapi-host`: `linkedin-api8.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key (replace placeholder).  
   - Connect Schedule Trigger output to this node input.

4. **Add a Split Out node:**  
   - Name: "Split Liked Posts"  
   - Field to split out: `data.items`  
   - Connect output of "Fetch LinkedIn Likes" to this node.

5. **Add a Filter node:**  
   - Name: "Filter Insightful & Recent"  
   - Conditions (AND):  
     - `$json.action` contains `"insightful"` (case-sensitive).  
     - `new Date($json.postedDate).getTime()` > `new Date().getTime() - (7 * 24 * 60 * 60 * 1000)` (posts within last 7 days).  
   - Connect output of "Split Liked Posts" to this node.

6. **Add a Set node:**  
   - Name: "Format Content Idea"  
   - Assign fields:  
     - `Title`: `I just liked a linkedin post of {{ $json.author.username }}`  
     - `description`: `{{ $json.text }}`  
     - `source`: `{{ $json.postUrl }}`  
   - Connect output of "Filter Insightful & Recent" to this node.

7. **Add another Split Out node:**  
   - Name: "Prepare for Airtable"  
   - Field to split out: `Title, description, source`  
   - Connect output of "Format Content Idea" to this node.

8. **Add an Airtable node:**  
   - Name: "Save to Airtable"  
   - Operation: Create  
   - Connect your Airtable account (Personal Access Token recommended).  
   - Select Base: `Content Hub`  
   - Select Table: `Ideas`  
   - Map columns:  
     - `Type`: Set fixed value `"Linkedin"`  
     - `Title`: `={{ $json.Title }}`  
     - `Source`: `={{ $json.source }}`  
     - `Status`: `false` (boolean)  
     - `Description`: `={{ $json.description }}`  
   - Connect output of "Prepare for Airtable" to this node.

9. **Activate the workflow** after verifying all connections and credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires a RapidAPI account with access to the `linkedin-api8` endpoint.                                                | https://rapidapi.com/                                                                              |
| Airtable base named `Content Hub` with a table `Ideas` must be created with columns: Title (single line), Description (long text), Source (URL), and Type (single select with value "Linkedin"). | https://airtable.com/                                                                              |
| The workflow automates content ideation by turning LinkedIn post reactions into actionable content ideas stored in Airtable.         | Workflow description and use case summary.                                                        |
| Ensure your RapidAPI key and LinkedIn username are correctly set in the HTTP Request node headers and query parameters respectively. | Critical for successful API calls.                                                                 |
| Airtable Personal Access Token credentials must be created and linked in n8n for the Airtable node to function properly.             | https://airtable.com/api                                                                           |
| For troubleshooting API errors, check API quotas, key validity, and network connectivity.                                             | Common integration issues.                                                                         |