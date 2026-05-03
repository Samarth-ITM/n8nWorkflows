Sync Zendesk Knowledge Base Articles to Airtable with Markdown Conversion

https://n8nworkflows.xyz/workflows/sync-zendesk-knowledge-base-articles-to-airtable-with-markdown-conversion-5876


# Sync Zendesk Knowledge Base Articles to Airtable with Markdown Conversion

### 1. Workflow Overview

This workflow is designed to synchronize Zendesk Knowledge Base articles into an Airtable base, converting article content from HTML to Markdown format for easier reuse and portability. It targets customer support teams, knowledge managers, and automation specialists who want to consolidate, back up, or repurpose Zendesk help center content in Airtable or other Markdown-friendly platforms.

The workflow is logically divided into these main blocks:

- **1.1 Triggers:** Manual and scheduled triggers to start the sync process either fully or incrementally.
- **1.2 Zendesk API Data Retrieval:** Fetches Zendesk articles via the Help Center API, supporting both full and incremental fetch modes, including pagination.
- **1.3 Data Preparation:** Splits the article array into individual items, extracts relevant fields, and converts article bodies from HTML to Markdown.
- **1.4 Data Storage:** Upserts articles into an Airtable base using the Article ID as the unique key to avoid duplicates.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers

**Overview:**  
This block initiates the workflow either manually (full sync) or automatically on a schedule (incremental sync).

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Configuration: No parameters; used for on-demand full sync execution.  
  - Inputs: None  
  - Outputs: Connected to "Set base_url" node to start full fetch.  
  - Edge cases: User must manually execute; no automatic retries.  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs every 30 days (configurable interval).  
  - Inputs: None  
  - Outputs: Connected to "Set base url" node to start incremental fetch.  
  - Edge cases: If the workflow is disabled or n8n is down, scheduled runs may be missed.  

---

#### 2.2 Zendesk API Data Retrieval

**Overview:**  
Fetches articles from Zendesk Help Center using two approaches: full fetch (all articles) and incremental fetch (articles updated since last run). Both use pagination to handle large data sets.

**Nodes Involved:**  
- Set base_url (for manual full fetch)  
- Get all articles (full fetch)  
- Set base url (for scheduled incremental fetch)  
- Get articles since last run (incremental fetch)

**Node Details:**

- **Set base_url (Manual Trigger branch)**  
  - Type: Set  
  - Configuration: Assigns `base_url` variable with Zendesk domain URL (e.g., `https://agentstudiohelp.zendesk.com`).  
  - Inputs: From Manual Trigger  
  - Outputs: To "Get all articles"  
  - Notes: User must edit this base_url before running.  
  - Edge cases: Incorrect URL or missing trailing slash can cause API failures.  

- **Get all articles**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `{{ $json.base_url }}/api/v2/help_center/articles`  
    - Pagination: Uses `next_page` URL from Zendesk response to paginate until all pages fetched.  
    - Query parameters: Sorted by `created_at` descending.  
    - Authentication: Zendesk API credentials (token-based).  
  - Inputs: From "Set base_url"  
  - Outputs: To "Split Out" node  
  - Edge cases: API rate limits, network timeouts, invalid credentials.  
  - Version: HTTP Request node v4.2 supports advanced pagination.  

- **Set base url (Schedule Trigger branch)**  
  - Type: Set  
  - Configuration: Same as above, assigns Zendesk domain URL to `base_url`.  
  - Inputs: From Schedule Trigger  
  - Outputs: To "Get articles since last run"  
  - Edge cases: Same as above.  

- **Get articles since last run**  
  - Type: HTTP Request  
  - Configuration:  
    - URL uses Zendesk incremental endpoint with `start_time` query parameter calculated as current time minus schedule interval in seconds (days converted to seconds).  
    - Pagination: Same pagination method as full fetch.  
    - Authentication: Zendesk API credentials.  
  - Inputs: From "Set base url" (schedule branch)  
  - Outputs: To "Split Out" node  
  - Edge cases: If schedule interval changes, may miss or duplicate articles; API limits; incorrect time calculation.  

---

#### 2.3 Data Preparation

**Overview:**  
Processes raw article data by splitting the array into individual articles, extracting relevant fields, and converting article bodies from HTML to Markdown.

**Nodes Involved:**  
- Split Out  
- Set fields to store  
- Body to Markdown

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits on JSON field `articles` to process each article separately.  
  - Inputs: From either "Get all articles" or "Get articles since last run"  
  - Outputs: To "Set fields to store"  
  - Edge cases: Empty or malformed `articles` array could cause no output or errors.  

- **Set fields to store**  
  - Type: Set  
  - Configuration: Extracts and renames fields:  
    - `id` (number)  
    - `url` (string)  
    - `title` (string)  
    - `body` (string, HTML content)  
  - Inputs: From "Split Out"  
  - Outputs: To "Body to Markdown"  
  - Edge cases: Missing fields in article JSON may cause undefined values.  

- **Body to Markdown**  
  - Type: Markdown  
  - Configuration: Converts `body` HTML content to Markdown, stored in `markdown` field.  
  - Inputs: From "Set fields to store"  
  - Outputs: To "Store Zendesk articles to Airtable"  
  - Edge cases: Complex HTML might not convert perfectly; empty bodies result in empty Markdown.  

---

#### 2.4 Data Storage

**Overview:**  
Upserts each processed article into Airtable, matching by `Article ID` to avoid duplicates and ensure updates.

**Nodes Involved:**  
- Store Zendesk articles to Airtable

**Node Details:**

- **Store Zendesk articles to Airtable**  
  - Type: Airtable node  
  - Configuration:  
    - Operation: Upsert (inserts or updates if exists)  
    - Base: Selected Airtable base (linked by ID)  
    - Table: Articles table  
    - Matching column: `Article ID`  
    - Fields mapped:  
      - URL → `url`  
      - Title → `title`  
      - Content → `markdown` (Markdown-converted body)  
      - Article ID → `id`  
  - Inputs: From "Body to Markdown"  
  - Outputs: None (end of workflow)  
  - Credentials: Airtable API token with read/write permissions  
  - Edge cases: Airtable API rate limits, invalid credentials, schema mismatches (columns missing or renamed).  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                     | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                         |
|--------------------------------|---------------------|-----------------------------------|--------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger      | Starts full sync manually          | None                           | Set base_url                  | Starts a full sync                                                                                |
| Set base_url                   | Set                 | Sets Zendesk domain URL for full fetch | When clicking ‘Execute workflow’ | Get all articles              | Sets your Zendesk domain. **💡 Edit the `base_url` before running**                               |
| Get all articles               | HTTP Request        | Fetches all Zendesk articles       | Set base_url                   | Split Out                     | Full Fetch: Get Articles - Pulls all articles with pagination                                     |
| Split Out                     | Split Out           | Splits articles array to individual items | Get all articles / Get articles since last run | Set fields to store           | Prepare the data - Splits array, selects fields, converts HTML to Markdown                        |
| Set fields to store            | Set                 | Extracts relevant article fields   | Split Out                     | Body to Markdown              | Prepare the data - selects and renames fields                                                    |
| Body to Markdown               | Markdown            | Converts article HTML body to Markdown | Set fields to store            | Store Zendesk articles to Airtable | Prepare the data - converts HTML to Markdown                                                     |
| Store Zendesk articles to Airtable | Airtable           | Upserts articles into Airtable     | Body to Markdown              | None                         | Store Articles - Upserts by Article ID; update Airtable base & table before running              |
| Schedule Trigger              | Schedule Trigger    | Starts incremental sync automatically | None                          | Set base url                  | Runs the workflow automatically on schedule - fetches new/updated articles since last run       |
| Set base url                  | Set                 | Sets Zendesk domain URL for incremental fetch | Schedule Trigger              | Get articles since last run   | Incremental sync                                                                                  |
| Get articles since last run   | HTTP Request        | Fetches articles updated since last run | Set base url                  | Split Out                    | Incremental Fetch: Get Articles since last run - pulls only new/updated articles with pagination |
| Sticky Note1                  | Sticky Note         | Overview and instructions          | None                          | None                         | Full detailed workflow overview and prerequisites                                                |
| Sticky Note2                  | Sticky Note         | Marks initial sync area            | None                          | None                         | Initial sync                                                                                      |
| Sticky Note3                  | Sticky Note         | Highlights manual trigger purpose  | None                          | None                         | Starts a full sync                                                                               |
| Sticky Note4                  | Sticky Note         | Reminder to set Zendesk domain URL | None                          | None                         | Sets your Zendesk domain. **💡 Edit the `base_url` before running**                              |
| Sticky Note5                  | Sticky Note         | Explains full fetch node           | None                          | None                         | Full Fetch: Get Articles - pulls all with pagination                                            |
| Sticky Note6                  | Sticky Note         | Explains incremental fetch node    | None                          | None                         | Incremental Fetch: Get Articles since last run                                                  |
| Sticky Note7                  | Sticky Note         | Explains schedule trigger          | None                          | None                         | Runs automatically on schedule - fetches only new/updated articles since last run               |
| Sticky Note8                  | Sticky Note         | Marks incremental sync area        | None                          | None                         | Incremental sync                                                                                  |
| Sticky Note9                  | Sticky Note         | Describes data preparation         | None                          | None                         | Prepare the data - splitting, field selection, HTML to Markdown conversion                       |
| Sticky Note10                 | Sticky Note         | Describes Airtable storage         | None                          | None                         | Store Articles - Upserts into Airtable, matching by Article ID                                  |
| Sticky Note12                 | Sticky Note         | Attribution and credits            | None                          | None                         | Created by Agent Studio - contact and template links                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters; used to manually start full sync.

2. **Create Set node**  
   - Name: `Set base_url`  
   - Assign variable `base_url` to your Zendesk domain URL (e.g., `https://agentstudiohelp.zendesk.com`).  
   - Connect output from Manual Trigger to this node.

3. **Create HTTP Request node**  
   - Name: `Get all articles`  
   - URL: `={{ $json.base_url }}/api/v2/help_center/articles`  
   - Method: GET  
   - Query parameters: `sort_by` = `created_at`, `sort_order` = `desc`  
   - Enable Pagination:  
     - Pagination mode: Response contains next URL  
     - Next URL path: `{{$response.body.next_page}}`  
     - Request interval: 1000 ms  
     - Complete expression: `{{$response.body.next_page.isEmpty()}}`  
   - Authentication: Set to Zendesk API credentials (use token access)  
   - Connect output from `Set base_url` to this node.

4. **Create Split Out node**  
   - Name: `Split Out`  
   - Field to split out: `articles`  
   - Connect output from `Get all articles`.

5. **Create Set node**  
   - Name: `Set fields to store`  
   - Assign fields:  
     - `id` = `={{ $json.id }}` (number)  
     - `url` = `={{ $json.url }}` (string)  
     - `title` = `={{ $json.title }}` (string)  
     - `body` = `={{ $json.body }}` (string)  
   - Connect output from `Split Out`.

6. **Create Markdown node**  
   - Name: `Body to Markdown`  
   - Input HTML: `={{ $json.body }}`  
   - Destination key: `markdown`  
   - Connect output from `Set fields to store`.

7. **Create Airtable node**  
   - Name: `Store Zendesk articles to Airtable`  
   - Operation: Upsert  
   - Base: Select your Airtable base (use the one from the provided template or your own)  
   - Table: Select `Articles` or equivalent  
   - Matching columns: `Article ID`  
   - Map fields:  
     - URL → `={{ $json.url }}`  
     - Title → `={{ $json.title }}`  
     - Content → `={{ $json.markdown }}`  
     - Article ID → `={{ $json.id }}`  
   - Credentials: Use Airtable API token with read/write permissions  
   - Connect output from `Body to Markdown`.

8. **Create Schedule Trigger node**  
   - Name: `Schedule Trigger`  
   - Set interval: every 30 days (or as desired)  
   - Connect output to next Set node for incremental fetch.

9. **Create Set node**  
   - Name: `Set base url` (for incremental fetch)  
   - Assign variable `base_url` same as in step 2  
   - Connect output from `Schedule Trigger`.

10. **Create HTTP Request node**  
    - Name: `Get articles since last run`  
    - URL:  
      ```
      ={{ $json.base_url }}/api/v2/help_center/incremental/articles.json?start_time={{ $now.minus($('Schedule Trigger').params.rule.interval.first().daysInterval, 'days').toSeconds().round() }}
      ```  
    - Method: GET  
    - Pagination: same as full fetch node  
    - Authentication: Zendesk API credentials  
    - Connect output from `Set base url`.

11. **Connect `Get articles since last run` to `Split Out` node** (same Split Out node as full fetch).

12. **Ensure all connections flow as:**  
    - Manual Trigger → Set base_url (manual) → Get all articles → Split Out → Set fields to store → Body to Markdown → Airtable upsert  
    - Schedule Trigger → Set base url (schedule) → Get articles since last run → Split Out → Set fields to store → Body to Markdown → Airtable upsert

13. **Create and configure credentials:**  
    - Zendesk API credentials with domain, email, and API token (token access enabled)  
    - Airtable API token with appropriate scopes (`data.records:read` and `data.records:write`)

14. **Test workflow:**  
    - Run manual trigger once to fetch all articles.  
    - Confirm Airtable base is updated as expected.  
    - Enable Schedule Trigger for incremental sync.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow fetches Zendesk Knowledge Base articles and syncs them to Airtable, converting HTML to Markdown for easier reuse.                                                                                                                                                                                          | Overview of workflow purpose                                                                                  |
| Airtable base template used: [Airtable Template](https://airtable.com/apptzJnbB6FphIprO/shrA6AhTkogTgrRn5) with fields: Title, Content, URL, Article ID.                                                                                                                                                               | Airtable base setup                                                                                           |
| Zendesk API token creation instructions: Admin Center > Apps and Integrations > Zendesk API, enable token access, create API token.                                                                                                                                                                                      | Zendesk API setup instructions                                                                                 |
| Airtable personal access token creation instructions: Account settings > Generate token with scopes `data.records:read` and `data.records:write`.                                                                                                                                                                        | Airtable API setup instructions                                                                                |
| API rate limits may affect syncing large numbers of articles; n8n handles pagination but long runs could hit limits.                                                                                                                                                                                                      | Operational considerations                                                                                      |
| No duplicates: The workflow uses Article ID as unique key to upsert and avoid duplicates in Airtable.                                                                                                                                                                                                                   | Data consistency note                                                                                           |
| Adaptable: Workflow can be modified to store Markdown content in other platforms like Notion, Google Sheets, or custom databases.                                                                                                                                                                                       | Extensibility note                                                                                              |
| Attribution: Workflow template created by Agent Studio, available free in n8n community. Contact: hello@agentstudio.io. More templates: https://n8n.io/creators/agentstudio/                                                                                                                                            | Attribution and support                                                                                         |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. All content processed is legal and publicly accessible, fully compliant with content policies.