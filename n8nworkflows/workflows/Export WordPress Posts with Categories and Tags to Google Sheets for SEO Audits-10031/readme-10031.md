Export WordPress Posts with Categories and Tags to Google Sheets for SEO Audits

https://n8nworkflows.xyz/workflows/export-wordpress-posts-with-categories-and-tags-to-google-sheets-for-seo-audits-10031


# Export WordPress Posts with Categories and Tags to Google Sheets for SEO Audits

### 1. Workflow Overview

This n8n workflow automates the extraction of WordPress post data, including associated categories and tags, and exports this enriched data into a Google Sheets document. It is designed primarily for SEO audits or content analysis where a structured overview of posts and their metadata is required.  

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a form trigger where the WordPress site URL and post limit are provided.
- **1.2 Configuration:** Sets internal parameters for API pagination limits.
- **1.3 Data Retrieval:** Fetches posts, categories, and tags from the WordPress REST API.
- **1.4 Data Aggregation:** Merges the fetched datasets into a unified stream.
- **1.5 Data Enrichment:** Maps category and tag IDs in posts to their human-readable names.
- **1.6 Data Export:** Appends the enriched post data into a specified Google Sheets document.
- **1.7 Completion Notification:** Sends a confirmation form response upon successful data export.
- **1.8 Error Handling:** Captures and reports WordPress API errors via a dedicated form.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects user inputs necessary for the workflow: WordPress site URL and the number of posts to export.
- **Nodes Involved:**  
  - On form submission
- **Node Details:**

  - **On form submission**  
    - **Type:** Form Trigger  
    - **Role:** Entry-point to capture URL (string, required) and Post limit (number, optional) from the user.  
    - **Configuration:**  
      - Form titled "WP SEO Audit"  
      - Fields:  
        - URL, placeholder "http://yourdomain.com", required  
        - Post limit, placeholder "Default =10" (optional)  
    - **Inputs:** HTTP request via webhook on form submission  
    - **Outputs:** JSON containing `URL` and `Post limit`  
    - **Edge cases:**  
      - Missing or invalid URL input (could cause downstream HTTP request failures)  
      - Post limit omitted (default handled downstream)  
    - **Version:** 2.3

#### 2.2 Configuration

- **Overview:** Sets static pagination parameter `per_page` used for fetching categories and tags.
- **Nodes Involved:**  
  - Config
- **Node Details:**

  - **Config**  
    - **Type:** Set node  
    - **Role:** Defines a workflow variable `per_page` with value 100, used to limit categories and tags fetched.  
    - **Configuration:** Assigns `per_page` = 100  
    - **Inputs:** Triggered after `On form submission` node  
    - **Outputs:** JSON with `per_page` property  
    - **Edge cases:** Hardcoded value; changing value requires manual update here.  
    - **Version:** 3.4

#### 2.3 Data Retrieval

- **Overview:** Requests posts, categories, and tags from the WordPress REST API based on inputs and config.
- **Nodes Involved:**  
  - Get Posts  
  - Get Categories  
  - Get Tags
- **Node Details:**

  - **Get Posts**  
    - **Type:** HTTP Request  
    - **Role:** Fetches posts from WordPress REST API endpoint `/wp-json/wp/v2/posts` with `per_page` limited by user input `Post limit`.  
    - **Configuration:** URL expression uses `{{$json.URL}}/wp-json/wp/v2/posts?per_page={{$json['Post limit']}}`  
    - **Error Handling:** On error, continue with error output to `WP API Error` node  
    - **Output:** Array of post JSON objects, including categories and tags as IDs  
    - **Edge Cases:**  
      - API unavailability or wrong URL leads to error  
      - Post limit not provided or invalid number  
      - Pagination limits imposed by WordPress API (max 100)  
    - **Version:** 4.2

  - **Get Categories**  
    - **Type:** HTTP Request  
    - **Role:** Fetches post categories from `/wp-json/wp/v2/categories` with `per_page` set to 100 from `Config` node.  
    - **Configuration:** URL expression uses `{{$('On form submission').item.json.URL}}/wp-json/wp/v2/categories?per_page={{$json.per_page}}`  
    - **Error Handling:** Same as Get Posts  
    - **Output:** Array of category JSON objects (id, name, etc.)  
    - **Edge Cases:** Same as Get Posts, plus possible empty categories list  
    - **Version:** 4.2

  - **Get Tags**  
    - **Type:** HTTP Request  
    - **Role:** Fetches post tags from `/wp-json/wp/v2/tags` with `per_page` set to 100 from `Config` node.  
    - **Configuration:** URL expression similar to Get Categories  
    - **Error Handling:** Same as above  
    - **Output:** Array of tag JSON objects  
    - **Edge Cases:** Same as Get Categories  
    - **Version:** 4.2

#### 2.4 Data Aggregation

- **Overview:** Combines the three separate datasets (posts, categories, tags) into one stream to process together.
- **Nodes Involved:**  
  - Merge
- **Node Details:**

  - **Merge**  
    - **Type:** Merge  
    - **Role:** Merges the outputs of Get Tags, Get Categories, and Get Posts nodes into a single unified collection for processing.  
    - **Configuration:** Number of inputs set to 3, merging all three datasets.  
    - **Inputs:**  
      - Input 1: Get Tags  
      - Input 2: Get Categories  
      - Input 3: Get Posts  
    - **Outputs:** Single combined output array maintaining order  
    - **Edge Cases:**  
      - One or more inputs empty or errored (handled by error continuation)  
    - **Version:** 3.2

#### 2.5 Data Enrichment

- **Overview:** Transforms post data by replacing category and tag IDs with their names for readability.
- **Nodes Involved:**  
  - Assign tags and categories names to posts
- **Node Details:**

  - **Assign tags and categories names to posts**  
    - **Type:** Code (JavaScript)  
    - **Role:**  
      - Reads the arrays of tags, categories, and posts.  
      - Builds lookup maps from IDs to names for tags and categories.  
      - For each post, replaces IDs arrays with arrays of corresponding names (`categoryNames`, `tagNames`).  
    - **Key expressions:**  
      - Uses `$items('Get Tags')`, `$items('Get Categories')`, `$items('Get Posts')` to access previous nodes’ data.  
      - Helper function `toIdArray` to normalize single or multiple IDs into arrays.  
    - **Inputs:** Merged data from previous step  
    - **Outputs:** Array of posts enriched with `categoryNames` and `tagNames` fields  
    - **Edge Cases:**  
      - Missing or null IDs handled gracefully.  
      - Posts with no categories or tags get empty arrays.  
      - Possible undefined mapping if categories/tags missing.  
    - **Version:** 2

#### 2.6 Data Export

- **Overview:** Appends enriched post data into a Google Sheet dedicated to SEO audits.
- **Nodes Involved:**  
  - Add posts, with tags, categories to Google Sheet
- **Node Details:**

  - **Add posts, with tags, categories to Google Sheet**  
    - **Type:** Google Sheets  
    - **Role:** Appends rows to a specified Google Sheets document with columns for URL, Title, Categories, and Tags.  
    - **Configuration:**  
      - Spreadsheet ID: `1nZ3WHn_QedbuNYlFVfPildBZETNYYJFT0yYFIgJy2Mg`  
      - Sheet: `gid=0` (Sheet1)  
      - Columns mapped:  
        - URL → `$json.link`  
        - Title → `$json.title.rendered`  
        - Categories → `$json.categoryNames`  
        - Tags → `$json.tagNames`  
      - Append operation (adds rows without overwriting)  
      - Credential: GoogleSheets OAuth2 API with user Piotr.Sikora.Ck@gmail.com  
    - **Inputs:** Enriched posts  
    - **Outputs:** Passes data to completion form  
    - **Edge Cases:**  
      - Google Sheets API errors (auth, quota, sheet missing)  
      - Data type mismatches (arrays converted to strings implicitly)  
      - Sheet columns must exist with exact names to prevent data loss  
    - **Version:** 4.7

#### 2.7 Completion Notification

- **Overview:** Sends a confirmation message to the user after successful data export.
- **Nodes Involved:**  
  - Form
- **Node Details:**

  - **Form**  
    - **Type:** Form (completion)  
    - **Role:** Displays a completion message to the user after workflow finishes.  
    - **Configuration:**  
      - Completion title: "List created"  
      - Message: "Please check linked document to see details"  
    - **Inputs:** From Google Sheets append node  
    - **Outputs:** None (terminal node)  
    - **Edge Cases:** Minimal, unless form webhook fails  
    - **Version:** 2.3

#### 2.8 Error Handling

- **Overview:** Captures and reports errors occurring during API fetches.
- **Nodes Involved:**  
  - WP API Error
- **Node Details:**

  - **WP API Error**  
    - **Type:** Form (completion)  
    - **Role:** Displays an error message if any of the HTTP requests to WordPress API fail.  
    - **Configuration:**  
      - Title: "WordPress API Error"  
      - Message: "Please check if your WP-Api is enabled"  
    - **Inputs:** From Get Posts, Get Categories, Get Tags on error outputs  
    - **Outputs:** None (terminal node)  
    - **Edge Cases:** Triggered if API endpoints are unreachable, disabled, or misconfigured  
    - **Version:** 2.3

---

### 3. Summary Table

| Node Name                               | Node Type           | Functional Role                                   | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                 |
|----------------------------------------|---------------------|--------------------------------------------------|------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------|
| On form submission                     | Form Trigger        | Receives user inputs (URL, Post limit)            |                              | Get Posts, Config                    |                                                                                             |
| Get Posts                             | HTTP Request        | Fetches posts from WP REST API                     | On form submission            | Merge, WP API Error                   |                                                                                             |
| Get Categories                       | HTTP Request        | Fetches categories from WP REST API                | Config                       | Merge, WP API Error                   |                                                                                             |
| Get Tags                             | HTTP Request        | Fetches tags from WP REST API                       | Config                       | Merge, WP API Error                   |                                                                                             |
| Merge                               | Merge               | Combines posts, categories, and tags datasets      | Get Tags, Get Categories, Get Posts | Assign tags and categories names to posts |                                                                                         |
| Assign tags and categories names to posts | Code                | Replaces category/tag IDs with their names in posts | Merge                        | Add posts, with tags, categories to Google Sheet |                                                                                         |
| Add posts, with tags, categories to Google Sheet | Google Sheets       | Appends enriched posts data into Google Sheets    | Assign tags and categories names to posts | Form                               | ## Add posts, with tags, categories to Google Sheet<br>Remember to create **Google Sheet** with fields:<br>- **URL**<br>- **Title**<br>- **Categories**<br>- **Tags** |
| Form                                | Form                | Displays completion message                         | Add posts, with tags, categories to Google Sheet |                                      |                                                                                             |
| WP API Error                        | Form                | Displays API error message                          | Get Posts, Get Categories, Get Tags (on error) |                                      | ## WP API Errror<br>API is not available                                                   |
| Config                              | Set                 | Sets fixed pagination parameter `per_page`         | On form submission           | Get Tags, Get Categories              | ## Config per_page<br>Configure `per_page` parameter                                        |
| Sticky Note                         | Sticky Note          | Notes on Google Sheets configuration                |                              |                                      | ## Add posts, with tags, categories to Google Sheet<br>Remember o create **Google Sheet** with fields:<br>- **URL**<br>- **Title**<br>- **Categories**<br>- **Tags** |
| Sticky Note1                        | Sticky Note          | Notes on WP API error                                |                              |                                      | ## WP API Errror<br>API is not avalable                                                    |
| Sticky Note2                        | Sticky Note          | Notes on API data fetching                           |                              |                                      | ## Fetch API data<br>Fetch the following resources:<br>- **Posts**<br>- **Categories**<br>- **Tags**<br><br>Please note that the `per_page` parameter is statically assigned for the posts and categories endpoints, with its value set to 100.<br><br>If you require a different number of results, adjust the parameter in the appropriate nodes accordingly. |
| Sticky Note3                        | Sticky Note          | Notes on data enrichment                             |                              |                                      | ## Append categories and tags names<br>What this does:<br>- Reads Get Tags, Get Categories, and Get Posts.<br>- Builds fast lookup maps.<br>- Appends `categoryNames` and `tagNames` directly into each post.<br>- Returns the modified posts ready for next steps. |
| Sticky Note4                        | Sticky Note          | Notes on configuration parameter `per_page`         |                              |                                      | ## Config per_page<br>Configure `per_page` parametter.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `On form submission`  
   - Type: Form Trigger (version 2.3)  
   - Configure form titled "WP SEO Audit" with fields:  
     - `URL`, type string, placeholder "http://yourdomain.com", required  
     - `Post limit`, type number, placeholder "Default =10", optional  
   - This node acts as the entry point.

2. **Create a Set Node for Configuration**  
   - Name: `Config`  
   - Type: Set (version 3.4)  
   - Add assignment: `per_page` = 100 (number)  
   - Connect the output of `On form submission` to `Config`.

3. **Create HTTP Request Nodes** to fetch WordPress data:  
   - **Get Posts**  
     - Name: `Get Posts`  
     - Type: HTTP Request (version 4.2)  
     - URL expression: `={{ $json.URL }}/wp-json/wp/v2/posts?per_page={{ $json['Post limit'] || 10 }}` (defaulting to 10 if undefined)  
     - Error handling: Set to continue on error output  
     - Connect output of `On form submission` to `Get Posts`.  

   - **Get Categories**  
     - Name: `Get Categories`  
     - Type: HTTP Request (version 4.2)  
     - URL expression: `={{ $json.URL }}/wp-json/wp/v2/categories?per_page={{ $json.per_page }}`  
     - Error handling: continue on error output  
     - Connect output of `Config` to `Get Categories`.  

   - **Get Tags**  
     - Name: `Get Tags`  
     - Type: HTTP Request (version 4.2)  
     - URL expression: `={{ $json.URL }}/wp-json/wp/v2/tags?per_page={{ $json.per_page }}`  
     - Error handling: continue on error output  
     - Connect output of `Config` to `Get Tags`.

4. **Create a Merge Node**  
   - Name: `Merge`  
   - Type: Merge (version 3.2)  
   - Set Number of Inputs to 3  
   - Connect outputs of `Get Tags`, `Get Categories`, and `Get Posts` to inputs 1, 2, and 3 respectively.

5. **Create a Code Node** for data enrichment  
   - Name: `Assign tags and categories names to posts`  
   - Type: Code (JavaScript, version 2)  
   - Paste the code that:  
     - Extracts tag and category arrays and builds ID-to-name maps  
     - Maps posts’ category and tag IDs to names arrays  
     - Returns enriched posts  
   - Connect output of `Merge` to this node.

6. **Create a Google Sheets Node**  
   - Name: `Add posts, with tags, categories to Google Sheet`  
   - Type: Google Sheets (version 4.7)  
   - Configure with appropriate Google Sheets OAuth2 credentials.  
   - Set spreadsheet ID to your target sheet (e.g., `1nZ3WHn_QedbuNYlFVfPildBZETNYYJFT0yYFIgJy2Mg`)  
   - Sheet name: default or `gid=0`  
   - Operation: Append  
   - Define columns mapping:  
     - URL → `{{$json.link}}`  
     - Title → `{{$json.title.rendered}}`  
     - Categories → `{{$json.categoryNames}}`  
     - Tags → `{{$json.tagNames}}`  
   - Connect output of code node to this node.

7. **Create a Form Node for Completion**  
   - Name: `Form`  
   - Type: Form (completion, version 2.3)  
   - Configure completion title: "List created"  
   - Configure completion message: "Please check linked document to see details"  
   - Connect output of Google Sheets node to this node.

8. **Create a Form Node for API Error Handling**  
   - Name: `WP API Error`  
   - Type: Form (completion, version 2.3)  
   - Configure completion title: "WordPress API Error"  
   - Configure completion message: "Please check if your WP-Api is enabled"  
   - Connect error outputs of `Get Posts`, `Get Categories`, and `Get Tags` nodes to this node.

9. **Add Sticky Notes (Optional but Recommended)** as per the original workflow to document important details such as Google Sheets field requirements, API error notes, and configuration instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Remember to create a Google Sheet with columns: URL, Title, Categories, Tags before running the workflow.                  | Sticky Note content in “Add posts, with tags, categories to Google Sheet” node                                      |
| WordPress REST API must be enabled and accessible on the target site for this workflow to function correctly.              | WP API Error form node and sticky note highlighting API errors                                                     |
| `per_page` parameter is statically set to 100 for categories and tags; adjust in Config node if different pagination needed. | Sticky Note about Config per_page parameter                                                                        |
| Posts limit is user-defined via form input; default to 10 if omitted.                                                      | Input Reception block and HTTP Request node for Get Posts                                                          |
| Google Sheets OAuth2 credentials must be configured with appropriate permissions to append data to the target spreadsheet. | Google Sheets node credential configuration                                                                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.