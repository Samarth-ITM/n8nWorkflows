Sync blog posts from Notion to Webflow

https://n8nworkflows.xyz/workflows/sync-blog-posts-from-notion-to-webflow-2293


# Sync blog posts from Notion to Webflow

### 1. Workflow Overview

This workflow automates the synchronization of blog posts maintained in a Notion database to a Webflow CMS collection. It is targeted at content managers, bloggers, or teams who use Notion to draft and manage blog entries and want a streamlined, once-daily sync process to publish or update posts on their Webflow-hosted website.

**Key Use Cases:**
- Daily syncing of Notion blog posts to Webflow.
- Handling rich text content including various heading levels, text stylings, lists, quotes, and images.
- Ensuring unique URL slugs for each post.
- Selective syncing based on a "Sync to Webflow?" checkbox in Notion.
- Creating new posts or updating existing ones on Webflow based on slug comparison.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Initial Data Retrieval:** Periodic trigger and fetching all blog posts from Notion.
- **1.2 Filtering & Slug Management:** Filtering posts to sync and ensuring slug uniqueness.
- **1.3 Detailed Notion Data Fetch & Cover Image Extraction:** Fetching all page data and extracting cover image URLs.
- **1.4 Rich Text Conversion:** Converting Notion content blocks into HTML suitable for Webflow rich text fields.
- **1.5 Webflow Data Comparison & Sync Decision:** Comparing Notion posts with Webflow collection items by slug to decide on create vs update.
- **1.6 Webflow Create or Update Operations:** Creating new posts or updating existing posts on Webflow.
- **1.7 Post-Sync Updates & Notifications:** Updating Notion posts with new slugs if needed and sending success notifications.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Initial Data Retrieval

**Overview:**  
This block initiates the workflow execution once a day and fetches all blog posts from the Notion database.

**Nodes Involved:**  
- Schedule Trigger  
- Get all blog posts1

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Initiates workflow on a daily schedule.  
  - Config: Default interval (daily).  
  - Inputs: None (trigger node).  
  - Outputs: Initiates downstream nodes.  
  - Edge cases: Trigger misconfiguration could disable the workflow.  
- **Get all blog posts1**  
  - Type: Notion Database Page GetAll  
  - Role: Retrieves all pages (blog posts) from the specified Notion database.  
  - Config: Uses Notion API credential, databaseId set to the blog database. Returns all entries without pagination limits.  
  - Inputs: Trigger output.  
  - Outputs: List of all blog posts in Notion.  
  - Edge cases: API rate limits, invalid databaseId, or credential expiration.

#### 1.2 Filtering & Slug Management

**Overview:**  
Filters the retrieved posts to only those marked for sync, then ensures unique URL slugs across posts.

**Nodes Involved:**  
- Is sync checked?1  
- Slug uniqueness checker and differentiator1  
- For each blog post1

**Node Details:**  

- **Is sync checked?1**  
  - Type: Filter  
  - Role: Keeps only posts where the "Sync to Webflow?" checkbox (property_sync_to_webflow) is true.  
  - Config: Boolean check on property_sync_to_webflow field.  
  - Inputs: Output from Get all blog posts1.  
  - Outputs: Filtered posts to sync.  
  - Edge cases: Missing or malformed checkbox property could exclude posts unintentionally.  
- **Slug uniqueness checker and differentiator1**  
  - Type: Code (JavaScript)  
  - Role: Ensures each post slug is unique by appending incremental numbers if duplicates exist.  
  - Config: Iterates all input items, tracks slug occurrences, modifies duplicates by appending "-n".  
  - Inputs: Filtered posts from Is sync checked?1.  
  - Outputs: Posts with unique slugs assigned.  
  - Edge cases: Slug missing or invalid format may cause issues; large datasets may impact performance.  
- **For each blog post1**  
  - Type: Split In Batches  
  - Role: Processes each post individually for detailed data fetching and sync operations.  
  - Config: Default batch size (likely 1), processes one post at a time.  
  - Inputs: Unique slug posts from previous node.  
  - Outputs: One post item per iteration.  
  - Edge cases: Large data volumes slow down processing.

#### 1.3 Detailed Notion Data Fetch & Cover Image Extraction

**Overview:**  
Fetches detailed data for each blog post, including rich content blocks and cover images.

**Nodes Involved:**  
- Get simple page data  
- Get all page data  
- Take cover url  
- Merge1

**Node Details:**  

- **Get simple page data**  
  - Type: Notion Database Page Get  
  - Role: Retrieves simplified page data for the current post ID.  
  - Config: Uses current post ID from batch item; simple mode enabled.  
  - Inputs: Output from For each blog post1 (branch 2).  
  - Outputs: Basic page data.  
  - Edge cases: Invalid ID, API errors.  
- **Get all page data**  
  - Type: Notion Database Page Get  
  - Role: Retrieves full page data with all properties for the current post ID.  
  - Config: Simple mode disabled to get rich data.  
  - Inputs: Output from For each blog post1 (branch 2).  
  - Outputs: Full page data.  
  - Edge cases: Large pages may cause timeouts or API limits.  
- **Take cover url**  
  - Type: Set  
  - Role: Extracts the cover image URL from the full page data (external URL assumed).  
  - Config: Assigns cover_url from json.cover.external.url property.  
  - Inputs: Output from Get all page data.  
  - Outputs: Data enriched with cover_url.  
  - Edge cases: Missing cover image or different cover type (file vs external).  
- **Merge1**  
  - Type: Merge  
  - Role: Combines simple page data and cover URL data into one enriched data item.  
  - Config: Merge by position (combine inputs in order).  
  - Inputs: From Get simple page data and Take cover url.  
  - Outputs: Single merged JSON object with all relevant page info.  
  - Edge cases: Misaligned inputs could cause data mismatch.

#### 1.4 Rich Text Conversion

**Overview:**  
Converts Notion content blocks into properly formatted HTML for Webflow rich text field compatibility, handling multiple block types.

**Nodes Involved:**  
- Get blocks1  
- Turn blocks into HTML1  
- Craft the rich text element1  
- Final Notion post data

**Node Details:**  

- **Get blocks1**  
  - Type: Notion Block GetAll  
  - Role: Retrieves all content blocks of the current blog post.  
  - Config: Uses current post ID, returns all blocks, output not simplified.  
  - Inputs: Output from Merge1.  
  - Outputs: Array of content blocks.  
  - Edge cases: Large number of blocks may affect performance.  
- **Turn blocks into HTML1**  
  - Type: Code (JavaScript)  
  - Role: Converts each block into an HTML snippet according to block type (headings, paragraphs, quotes, lists, images, code).  
  - Config: Handles annotations for text (bold, italic, links), and special elements like images with captions. Also adds fallback to div for unknown types.  
  - Inputs: Blocks from Get blocks1.  
  - Outputs: Array of objects containing block ID, type, and generated HTML.  
  - Edge cases: Malformed blocks or unsupported block types.  
- **Craft the rich text element1**  
  - Type: Code (JavaScript)  
  - Role: Aggregates the HTML snippets into a single rich text string, wrapping bulleted and numbered list items into `<ul>` and `<ol>` containers respectively.  
  - Config: Collects blocks of list items and emits them as grouped lists, appending other HTML directly.  
  - Inputs: Output from Turn blocks into HTML1.  
  - Outputs: Single object with property `newRichText` containing the full HTML string.  
  - Edge cases: Empty or incomplete block sets.  
- **Final Notion post data**  
  - Type: Merge  
  - Role: Combines the crafted rich text with the original Notion data for the post to prepare for Webflow sync.  
  - Config: Merge by position.  
  - Inputs: Craft the rich text element1 and prior data streams.  
  - Outputs: Complete post data with rich text HTML.

#### 1.5 Webflow Data Comparison & Sync Decision

**Overview:**  
Fetches all current posts from Webflow, compares by slug to determine if a post should be created or updated.

**Nodes Involved:**  
- Data transporter, Notion posts to sync1  
- Get all collection posts1  
- Compare by slug1  
- Add Webflow item id to Notion data

**Node Details:**  

- **Data transporter, Notion posts to sync1**  
  - Type: NoOp (pass-through)  
  - Role: Passes the final Notion post data to the comparison sequence.  
  - Inputs: Final Notion post data output.  
  - Outputs: Same data forwarded.  
- **Get all collection posts1**  
  - Type: Webflow GetAll  
  - Role: Retrieves all items from the Webflow blog collection to compare existing posts.  
  - Config: Uses siteId and collectionId for blog posts, returns all items.  
  - Inputs: Output from Data transporter node.  
  - Outputs: List of existing Webflow posts.  
  - Edge cases: API rate limits, credential issues.  
- **Compare by slug1**  
  - Type: Compare Datasets  
  - Role: Compares Notion posts and Webflow posts by slug to detect new vs existing posts.  
  - Config: Uses property_slug from Notion and slug from Webflow as keys.  
  - Inputs: Merged Notion post data and Webflow collection data.  
  - Outputs:  
    - "A only" branch: Posts present only in Notion (to create).  
    - "Different" branch: Posts present in both (to update).  
    - "B only" branch: Posts only in Webflow (not used here).  
  - Edge cases: Slug mismatches or duplicates could cause errors.  
- **Add Webflow item id to Notion data**  
  - Type: Code (JavaScript)  
  - Role: Injects the Webflow item ID into the Notion data for update operations.  
  - Config: Maps the Webflow _id to the Notion post data for downstream update use.  
  - Inputs: Output from "Different" branch of Compare by slug1 and Final Notion post data.  
  - Outputs: Enriched data with webflow_item_id.  
  - Edge cases: Missing or invalid IDs.

#### 1.6 Webflow Create or Update Operations

**Overview:**  
Creates new posts or updates existing posts on Webflow CMS based on comparison results.

**Nodes Involved:**  
- Create post1  
- Update in "Blog Posts"  
- Merge2  
- Add slug to posts1  
- Update slug on posts1

**Node Details:**  

- **Create post1**  
  - Type: Webflow Create  
  - Role: Creates a new post in Webflow using Notion post data.  
  - Config: Maps fields: name, slug, rich text content, cover images; marks post as draft and not archived. Uses OAuth2 credentials.  
  - Inputs: "A only" branch from Compare by slug1.  
  - Outputs: Created post data forwarded to Merge2.  
  - Edge cases: API errors, invalid field mappings, image size limitations.  
- **Update in "Blog Posts"**  
  - Type: Webflow Update  
  - Role: Updates an existing Webflow post with new data from Notion.  
  - Config: Uses webflow_item_id for itemId, updates same fields as creation node.  
  - Inputs: Output from Add Webflow item id to Notion data.  
  - Outputs: Updated post data.  
  - Edge cases: Item not found, stale IDs, API failures.  
- **Merge2**  
  - Type: Merge  
  - Role: Combines newly created posts and updated posts back into a single stream.  
  - Inputs: Create post1 (new items) and Update in "Blog Posts" (updated items).  
  - Outputs: All synced posts.  
- **Add slug to posts1**  
  - Type: Notion Page Update  
  - Role: Updates the slug property in Notion for newly created posts to keep data consistent.  
  - Inputs: Output from Merge2.  
  - Outputs: Updated Notion page data.  
  - Edge cases: API errors or permission issues.  
- **Update slug on posts1**  
  - Type: Notion Page Update  
  - Role: Updates slug property on posts that were updated on Webflow (in case of slug changes).  
  - Inputs: Output from Update in "Blog Posts".  
  - Outputs: Updated Notion page data.  
  - Edge cases: Same as Add slug to posts1.

#### 1.7 Post-Sync Updates & Notifications

**Overview:**  
Finalizes the sync by dispatching success notifications and optionally logging or displaying success messages.

**Nodes Involved:**  
- Success message1

**Node Details:**  

- **Success message1**  
  - Type: Slack  
  - Role: Sends a success message to a Slack channel indicating a post was successfully synced.  
  - Config: Uses OAuth2 credentials, posts to a fixed channel with post name in message text.  
  - Inputs: Output from For each blog post1 (success branch).  
  - Outputs: None (notification).  
  - Edge cases: Slack API rate limits or credential expiration.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                    | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                                                             |
|-----------------------------------|----------------------------|---------------------------------------------------|----------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                  | scheduleTrigger            | Starts workflow daily                             | None                             | Get all blog posts1                   |                                                                                                                                         |
| Get all blog posts1               | notion (databasePage getAll) | Retrieves all blog posts from Notion database    | Schedule Trigger                 | Is sync checked?1                     |                                                                                                                                         |
| Is sync checked?1                 | filter                    | Filters posts to only those checked for syncing | Get all blog posts1              | Slug uniqueness checker and differentiator1 |                                                                                                                                         |
| Slug uniqueness checker and differentiator1 | code                      | Ensures unique slugs across posts                 | Is sync checked?1                | For each blog post1                   |                                                                                                                                         |
| For each blog post1               | splitInBatches            | Processes each post individually                   | Slug uniqueness checker and differentiator1 | Success message1, Get simple page data, Get all page data |                                                                                                                                         |
| Get simple page data             | notion (databasePage get)  | Fetches simple page data                           | For each blog post1              | Merge1                              |                                                                                                                                         |
| Get all page data                | notion (databasePage get)  | Fetches full page data                             | For each blog post1              | Take cover url                      |                                                                                                                                         |
| Take cover url                   | set                       | Extracts cover image URL                           | Get all page data                | Merge1                              | "No wastes: These nodes extract the cover image url of the Notion page to make it easy for you to use it in the collection fields."     |
| Merge1                          | merge                     | Combines simple page data and cover URL           | Get simple page data, Take cover url | Get blocks1                       |                                                                                                                                         |
| Get blocks1                     | notion (block getAll)      | Retrieves all content blocks of post               | Merge1                          | Turn blocks into HTML1               |                                                                                                                                         |
| Turn blocks into HTML1           | code                      | Converts Notion blocks to HTML                      | Get blocks1                    | Craft the rich text element1         | "Turn blocks into rich text: This is where the magic happens — Notion blocks are mapped and turned into their respective html version..."|
| Craft the rich text element1     | code                      | Aggregates HTML snippets into full rich text string| Turn blocks into HTML1          | Final Notion post data               |                                                                                                                                         |
| Final Notion post data           | merge                     | Merges rich text with original Notion post data   | Craft the rich text element1, other | Data transporter, Notion posts to sync1 |                                                                                                                                         |
| Data transporter, Notion posts to sync1 | noOp                      | Passes Notion post data forward                    | Final Notion post data          | Get all collection posts1            |                                                                                                                                         |
| Get all collection posts1        | webflow (getAll)          | Retrieves all posts from Webflow collection        | Data transporter, Notion posts to sync1 | Compare by slug1                 |                                                                                                                                         |
| Compare by slug1                 | compareDatasets           | Compares Notion and Webflow posts by slug          | Get all collection posts1, Data transporter, Notion posts to sync1 | Create post1, Merge2, Add Webflow item id to Notion data | "Create a new post or update an existing one? This node compares (by slug) your Notion post with all your Webflow posts..."           |
| Create post1                    | webflow (create)          | Creates new post in Webflow                         | Compare by slug1 (A only)       | Merge2                             |                                                                                                                                         |
| Merge2                         | merge                     | Combines newly created and updated posts           | Create post1, Compare by slug1 (different branch) | Add slug to posts1                 |                                                                                                                                         |
| Add slug to posts1              | notion (update)            | Updates Notion post slug after creation            | Merge2                         | Data transporter1                   |                                                                                                                                         |
| Data transporter1               | noOp                      | Passes updated Notion post data                     | Add slug to posts1, Update slug on posts1 | For each blog post1              |                                                                                                                                         |
| Update slug on posts1           | notion (update)            | Updates Notion post slug after update               | Update in "Blog Posts"          | Data transporter1                   |                                                                                                                                         |
| Update in "Blog Posts"          | webflow (update)          | Updates existing Webflow post                        | Add Webflow item id to Notion data | Update slug on posts1             |                                                                                                                                         |
| Add Webflow item id to Notion data | code                      | Adds Webflow item ID to Notion post data            | Compare by slug1 (different branch), Final Notion post data | Update in "Blog Posts"           |                                                                                                                                         |
| Success message1                | slack                      | Sends Slack notification on successful sync        | For each blog post1 (success branch) | None                            | "Success: Send a success message where you want. You can remove this node."                                                           |
| Sticky Note                    | stickyNote                 | Installation instructions and setup notes           | None                          | None                              | "## Workflow installation\n* Add a \"slug\" text property to each blog post ... [image link]"                                            |
| Sticky Note1                   | stickyNote                 | Note about cover image extraction                    | None                          | None                              | "No wastes: These nodes extract the cover image url of the Notion page to make it easy for you to use it in the collection fields."     |
| Sticky Note17                  | stickyNote                 | Rich text block conversion explanation               | None                          | None                              | "Turn blocks into rich text: This is where the magic happens — Notion blocks are mapped and turned into their respective html version."|
| Sticky Note18                  | stickyNote                 | Create vs update decision explanation                 | None                          | None                              | "Create a new post or update an existing one? This node compares (by slug) your Notion post with all your Webflow posts ..."            |
| Sticky Note19                  | stickyNote                 | Success message explanation                            | None                          | None                              | "Success: Send a success message where you want. You can remove this node."                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run once daily (default).  
   - No credentials needed.

2. **Add a Notion Node: Get All Blog Posts**  
   - Type: Notion (Database Page GetAll)  
   - Operation: Get All from Database  
   - Database ID: Enter your Notion blog database ID.  
   - Credentials: Connect your Notion API credentials.  
   - Return All: true.

3. **Add a Filter Node: Is sync checked?**  
   - Type: Filter  
   - Condition: property_sync_to_webflow equals true (Boolean).  
   - Input: Connect from Get All Blog Posts.

4. **Add a Code Node: Slug uniqueness checker and differentiator**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that appends incremental numbers to duplicate slugs.  
   - Input: Connect from Filter node.

5. **Add a SplitInBatches Node: For each blog post**  
   - Type: SplitInBatches  
   - Batch Size: Default (1) to process posts individually.  
   - Input: Connect from Code node.

6. **Branch 1 for each blog post: Fetch detailed Notion data**  
   - Add two Notion nodes:  
     - Get simple page data (Database Page Get, simple mode)  
     - Get all page data (Database Page Get, simple mode disabled)  
   - Both use current batch post ID.  
   - Connect their outputs into a Merge node (Merge1), mode: Combine by position.

7. **Add a Set Node: Take cover url**  
   - Extract cover_url from the full page data (e.g., json.cover.external.url).  
   - Connect output to Merge1.

8. **Add a Notion Block GetAll Node: Get blocks**  
   - Use current post ID to get all blocks of the post.  
   - Connect from Merge1.

9. **Add a Code Node: Turn blocks into HTML**  
   - Paste the JavaScript converting Notion blocks to respective HTML snippets.  
   - Connect from Get blocks.

10. **Add a Code Node: Craft the rich text element**  
    - Paste the JavaScript combining block HTML snippets into a single rich text string, wrapping lists properly.  
    - Connect from previous Code node.

11. **Add a Merge Node: Final Notion post data**  
    - Merge the crafted rich text with prior post data.  
    - Connect from Craft the rich text element and other relevant data streams.

12. **Add a NoOp Node: Data transporter, Notion posts to sync**  
    - Pass through the merged post data.  
    - Connect from Final Notion post data.

13. **Add a Webflow GetAll Node: Get all collection posts**  
    - Retrieve all current posts from your Webflow blog collection.  
    - Provide siteId and collectionId.  
    - Connect from Data transporter node.  
    - Credentials: OAuth2 Webflow account.

14. **Add a Compare Datasets Node: Compare by slug**  
    - Compare Notion posts and Webflow posts by slug (property_slug vs slug).  
    - Connect Notion data and Webflow data inputs accordingly.  
    - Outputs three branches: "A only" (new posts), "Different" (existing posts to update), "B only" (not used).

15. **Create post branch:**  
    - Add a Webflow Create Node: Create post  
      - Map fields: name, slug, rich text, cover images, draft true, archived false.  
      - Connect from "A only" output.  
      - Credentials: Webflow OAuth2.  
    - Connect output to a Merge node (Merge2).

16. **Update post branch:**  
    - Add a Code Node: Add Webflow item id to Notion data  
      - Inject webflow_item_id from Webflow post (_id) into Notion data.  
      - Connect from "Different" output and Final Notion post data.  
    - Add a Webflow Update Node: Update in "Blog Posts"  
      - Use webflow_item_id as itemId.  
      - Map same fields as Create node.  
      - Credentials: Webflow OAuth2.  
      - Connect from Code node.  
    - Connect output to Merge2.

17. **Add a Merge Node: Merge2**  
    - Combine outputs from Create post and Update in "Blog Posts" nodes.  
    - Connect output to Notion update nodes.

18. **Add Notion Update Nodes:**  
    - Add slug to posts node: Updates slug property for newly created posts.  
    - Update slug on posts node: Updates slug property for updated posts.  
    - Connect Merge2 output to Add slug to posts; connect Update in "Blog Posts" output to Update slug on posts.  
    - Both update Notion pages with the current slug to ensure consistency.

19. **Connect Notion update outputs to Data transporter1 node**  
    - Pass data back to the For each blog post node for next iteration.

20. **Add a Slack Node: Success message**  
    - Sends notification on successful sync.  
    - Set message text to indicate the synced post name.  
    - Connect from For each blog post success branch.  
    - Credentials: Slack OAuth2.

21. **Add Sticky Notes**  
    - For installation instructions, rich text conversion explanation, and sync decision explanation, add Sticky Note nodes with the provided text and images for user reference.

22. **Configure Credentials:**  
    - Notion API credentials with appropriate scopes.  
    - Webflow OAuth2 credentials with CMS write access.  
    - Slack OAuth2 credentials for notification channel.

23. **Final Testing and Execution:**  
    - Ensure "slug" and "Sync to Webflow?" checkbox properties exist in Notion database.  
    - Map Webflow collection fields correctly during initial test runs.  
    - Run the workflow manually to verify full end-to-end sync.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Add a "slug" text property to each blog post in Notion and a "Sync to Webflow?" checkbox to control which posts are synced.                                                                                                                                          | Installation instructions from Sticky Note node with image: https://postimg.cc/BLbbxpJp               |
| The workflow supports most common Notion rich text elements, including headings, bold/italic text, links, quotes, bulleted and numbered lists, images under 4MB, and code blocks.                                                                                             | Sticky Note explanation near rich text conversion nodes                                               |
| When syncing, posts are added as drafts in Webflow and marked not archived, allowing preview before publishing.                                                                                                                                                         | Webflow create and update nodes configuration                                                         |
| Slack notifications help track successful syncs but can be removed if undesired.                                                                                                                                                                                           | Slack success message node                                                                             |
| The workflow ensures slug uniqueness by appending incremental suffixes to duplicate slugs before creating or updating Webflow items.                                                                                                                                    | Slug uniqueness checker and differentiator node                                                       |
| For blog posts with cover images, only external URLs are currently extracted for Webflow image fields; file-based covers may require adjustment.                                                                                                                            | Take cover url and Sticky Note about cover images                                                     |
| Webflow API usage includes retry on failure for robustness, but API limits or credential expiration can still cause errors.                                                                                                                                               | Webflow nodes configured with retryOnFail=true                                                        |

---

This completes the comprehensive reference documentation for the "Sync blog posts from Notion to Webflow" workflow, enabling advanced users and automation agents to understand, reproduce, and extend the solution effectively.