Generate SEO-Optimized Blog Posts with Google Autocomplete & GPT-4

https://n8nworkflows.xyz/workflows/generate-seo-optimized-blog-posts-with-google-autocomplete---gpt-4-6283


# Generate SEO-Optimized Blog Posts with Google Autocomplete & GPT-4

---

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog posts using Google Autocomplete, People Also Ask (PAA) data from SerpAPI, and GPT-4. It is designed for businesses or individuals who want to efficiently create relevant, keyword-rich blog content based on inspiration topics listed in a Google Sheet.

The logical structure is divided into the following blocks:

- **1.1 Input Reception & Filtering:** Watches a Google Sheet for new rows, reads them, and filters for rows without a "done" status.
- **1.2 Keyword Extraction & External Data Retrieval:** Extracts broad topic keywords, fetches Google Autocomplete suggestions and PAA data from SerpAPI, then merges these insights.
- **1.3 Content Generation Loop:** Iterates over each enriched topic, uses GPT-4 to generate a blog post based on the merged SEO data, and updates the Google Sheet with the finished post and status.
- **1.4 Export & Status Update:** Updates the original Google Sheet row marking the blog post as done and saving the generated content.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:** This block triggers the workflow upon new Google Sheets row additions, reads all rows, and filters for those where the "Status" column is empty or missing. This ensures only new or pending blog inspirations are processed.
- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Read Rows  
  - Only Reads Empty Status  
  - Use Wait Node for Large Batches  

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type & Role:* Trigger node that activates the workflow when a new row is added to a specified Google Sheet.  
    - *Configuration:* Monitors a specific document and sheet (configured via credentials and parameters).  
    - *Inputs/Outputs:* No inputs; outputs the newly added row data.  
    - *Failure Modes:* Authentication errors, Google API rate limiting, incorrect document or sheet IDs.  
    - *Notes:* Requires OAuth2 credentials for Google Sheets.

  - **Read Rows**  
    - *Type & Role:* Reads all rows from the Google Sheet to get a full data snapshot.  
    - *Configuration:* Uses same document and sheet as trigger; executes once per trigger.  
    - *Inputs/Outputs:* Input from trigger; outputs all rows.  
    - *Failure Modes:* API errors, permission issues.

  - **Only Reads Empty Status**  
    - *Type & Role:* Code node filtering rows to only those without a "done" status.  
    - *Configuration:* Filters items where `Status` field is undefined, empty, or blank.  
    - *Key Expression:* `item.json.Status?.trim()` and filter logic.  
    - *Inputs/Outputs:* Input all rows; output filtered rows.  
    - *Failure Modes:* Missing or malformed data fields.

  - **Use Wait Node for Large Batches**  
    - *Type & Role:* Wait node to throttle execution when processing large numbers of rows.  
    - *Configuration:* Default wait parameters; acts as a pacing mechanism.  
    - *Inputs/Outputs:* Input filtered rows; outputs after wait.  
    - *Failure Modes:* None typical; useful to avoid API throttling.

---

#### 2.2 Keyword Extraction & External Data Retrieval

- **Overview:** Extracts concise broad topic keywords from the "Blog Inspiration" column, then fetches Google Autocomplete suggestions and PAA question data from two external sources: a custom SEO API and SerpAPI. Results are merged for later use.
- **Nodes Involved:**  
  - Broad Words  
  - Autocomplete  
  - PAA (SerpAPI)  
  - Format PAA (SerpAPI)  
  - Merge  
  - All of the Information  

- **Node Details:**

  - **Broad Words**  
    - *Type & Role:* Code node that extracts the last three words from the "Blog Inspiration" text as a short topic keyword.  
    - *Configuration:* Splits the inspiration string and joins last three words; fallback keyword is "photo ideas".  
    - *Inputs/Outputs:* Input filtered rows; outputs items enriched with `topic` field.  
    - *Failure Modes:* Missing or empty inspiration text.

  - **Autocomplete**  
    - *Type & Role:* HTTP Request node invoking a custom SEO API via POST to get Google Autocomplete and PAA data.  
    - *Configuration:* Sends JSON body with the topic field to `https://seo-api2.onrender.com/get-seo-data`. Retries 5 times with 5 sec delay on failure.  
    - *Inputs/Outputs:* Input: items with `topic`; Output: API response with autocomplete + PAA data.  
    - *Failure Modes:* API downtime (service hosted on Render’s free tier may sleep), network errors, invalid responses, rate limiting.

  - **PAA (SerpAPI)**  
    - *Type & Role:* HTTP Request node querying SerpAPI for additional PAA data via Google search results.  
    - *Configuration:* Sends GET request with query parameters including the blog inspiration topic and API key.  
    - *Inputs/Outputs:* Input filtered rows; outputs raw SerpAPI JSON data.  
    - *Failure Modes:* Invalid or expired API key, request quota exceeded, network errors.

  - **Format PAA (SerpAPI)**  
    - *Type & Role:* Code node that parses SerpAPI JSON results to extract PAA questions, organic results, answer box content, or related searches as fallback.  
    - *Configuration:* Prioritizes related questions, then organic results, answer box, and related searches; formats them into strings with source tags (e.g., `(PAA)`, `(Organic)`).  
    - *Inputs/Outputs:* Input SerpAPI response; output cleaned, formatted PAA list per topic.  
    - *Failure Modes:* Unexpected JSON structure, missing fields.

  - **Merge**  
    - *Type & Role:* Merge node combining Autocomplete data and formatted PAA data streams by index.  
    - *Configuration:* Defaults to merge mode with no special parameters.  
    - *Inputs/Outputs:* Inputs from Autocomplete and Format PAA; output combined data objects.  
    - *Failure Modes:* Mismatched input lengths; missing data.

  - **All of the Information**  
    - *Type & Role:* Code node consolidating Autocomplete, SerpAPI PAA, and broad topic data into unified objects per topic.  
    - *Configuration:* Matches topics by normalized lowercase string; merges autocomplete suggestions and PAA questions, filters out irrelevant entries (e.g., "no paa").  
    - *Inputs/Outputs:* Input merged data; outputs array of objects containing `topic`, `autocomplete` array, and combined `paa` array or undefined if empty.  
    - *Failure Modes:* Index mismatch, null or undefined data fields.

---

#### 2.3 Content Generation Loop

- **Overview:** Iterates over each consolidated topic data item and uses GPT-4 via LangChain nodes to generate a warm, SEO-optimized blog post based on autocomplete keywords, PAA data, and blog inspiration.  
- **Nodes Involved:**  
  - Loop Over Items  
  - Generate Blog Post  
  - GPT-4  
  - Update "Done" Status  

- **Node Details:**

  - **Loop Over Items**  
    - *Type & Role:* SplitInBatches node that processes one topic item at a time to prevent API overload and ensure ordered processing.  
    - *Configuration:* Default batching options; processes items sequentially.  
    - *Inputs/Outputs:* Input consolidated topic data; outputs single item per iteration.  
    - *Failure Modes:* Large batch size causing API rate limits.

  - **Generate Blog Post**  
    - *Type & Role:* LangChain Agent node configured to generate blog content using OpenAI’s GPT-4 model.  
    - *Configuration:* Uses a detailed prompt instructing GPT-4 to create a warm, engaging, SEO-optimized ~500 word blog post with natural keyword integration and question answering. No headings unless structural. Ends with a call-to-action.  
    - *Inputs/Outputs:* Input topic object with `topic`, `autocomplete`, and `paa`; outputs generated blog content.  
    - *Failure Modes:* API key limits, model unavailability, prompt formatting errors.

  - **GPT-4**  
    - *Type & Role:* LangChain LM Chat OpenAI node using GPT-4 to finalize or process language model tasks.  
    - *Configuration:* Model set to `gpt-4o` (GPT-4 with optimized parameters).  
    - *Inputs/Outputs:* Receives prompt from Generate Blog Post node; outputs generated text.  
    - *Failure Modes:* OpenAI API quota, auth errors.

  - **Update "Done" Status**  
    - *Type & Role:* Set node updating the current item with the generated blog post in "Blog Draft" and marking "Status" as "done".  
    - *Configuration:* Assigns three properties: `"Blog Draft "` with generated output, `"Blog Inspiration"` with topic, and `"Status"` set to `"done"`.  
    - *Inputs/Outputs:* Input generated content; output updated item.  
    - *Failure Modes:* Data mismatch or missing fields.

---

#### 2.4 Export & Status Update

- **Overview:** Updates the original Google Sheet by writing back the generated blog post and marking the status to "done" to prevent re-processing.  
- **Nodes Involved:**  
  - Export  

- **Node Details:**

  - **Export**  
    - *Type & Role:* Google Sheets node performing an update operation on the row corresponding to the processed topic.  
    - *Configuration:* Uses OAuth2 credentials; updates columns in the sheet with new content and status.  
    - *Inputs/Outputs:* Input updated items with blog draft and status; no further output.  
    - *Failure Modes:* API errors, permission issues, row mismatch.

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                              | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                            |
|-----------------------|-------------------------------|----------------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger  | googleSheetsTrigger            | Triggers workflow on new sheet row           | —                           | Read Rows                  | ## Entering the Data: Triggered on new rows, processes only rows without "done" status.                                                                |
| Read Rows             | googleSheets                  | Reads all rows from Google Sheet              | Google Sheets Trigger        | Only Reads Empty Status     |                                                                                                                                                        |
| Only Reads Empty Status| code                          | Filters rows without "done" status            | Read Rows                   | Use Wait Node for Large Batches |                                                                                                                                                        |
| Use Wait Node for Large Batches | wait                  | Adds delay to throttle batch processing       | Only Reads Empty Status      | Broad Words, PAA (SerpAPI)  |                                                                                                                                                        |
| Broad Words           | code                          | Extracts last 3 words as topic keywords       | Use Wait Node for Large Batches | Autocomplete            | ## Google Autocomplete and PAA: Extracts broad keywords for Autocomplete and PAA queries.                                                              |
| Autocomplete          | httpRequest                   | Fetches Google Autocomplete + PAA from custom SEO API | Broad Words               | Merge                      | Uses https://seo-api2.onrender.com/get-seo-data; hosted on Render, may be slow on first request after idle.                                            |
| PAA (SerpAPI)         | httpRequest                   | Fetches additional PAA data via SerpAPI       | Use Wait Node for Large Batches | Format PAA (SerpAPI)     | Requires SerpAPI API key.                                                                                                                              |
| Format PAA (SerpAPI)  | code                          | Parses and formats SerpAPI PAA JSON data      | PAA (SerpAPI)               | Merge                      |                                                                                                                                                        |
| Merge                 | merge                         | Combines Autocomplete and PAA data            | Autocomplete, Format PAA (SerpAPI) | All of the Information  |                                                                                                                                                        |
| All of the Information| code                          | Merges all SEO data per topic                  | Merge                       | Loop Over Items             |                                                                                                                                                        |
| Loop Over Items       | splitInBatches                | Processes topics sequentially for content generation | All of the Information    | Generate Blog Post          | ## ChatGPT-4 and Export: Loops over each topic to generate blog posts and update status.                                                              |
| Generate Blog Post    | langchain.agent               | Generates SEO-optimized blog posts with GPT-4| Loop Over Items             | Update "Done" Status        |                                                                                                                                                        |
| GPT-4                 | langchain.lmChatOpenAi        | Language Model node for GPT-4                   | Generate Blog Post           | —                          |                                                                                                                                                        |
| Update "Done" Status  | set                           | Sets blog draft content and marks status done | Generate Blog Post          | Export                     |                                                                                                                                                        |
| Export                | googleSheets                  | Updates Google Sheet row with blog draft and status | Update "Done" Status       | —                          |                                                                                                                                                        |
| Sticky Note           | stickyNote                    | Documentation note                             | —                           | —                          | ## Entering the Data: Explains input reception and filtering.                                                                                         |
| Sticky Note1          | stickyNote                    | Documentation note                             | —                           | —                          | ## Google Autocomplete and PAA: Explains data sources and merging process.                                                                             |
| Sticky Note2          | stickyNote                    | Documentation note                             | —                           | —                          | ## ChatGPT-4 and Export: Explains content generation and exporting logic.                                                                              |
| Sticky Note3          | stickyNote                    | Documentation note                             | —                           | —                          | ## Example Input: Visual preview of input data.                                                                                                       |
| Sticky Note4          | stickyNote                    | Documentation note                             | —                           | —                          | ## Example Output: Visual preview of generated blog post.                                                                                              |
| Sticky Note5          | stickyNote                    | Documentation note                             | —                           | —                          | ## Try It Out! Full project description, usage instructions, and setup notes including links and API considerations.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Type: `googleSheetsTrigger`  
   - Configure with OAuth2 credentials for Google Sheets.  
   - Set event to `rowAdded` on the desired document and sheet.

2. **Add Google Sheets node to read rows:**  
   - Type: `googleSheets`  
   - Configure to read all rows from the same document and sheet.  
   - Connect output of Google Sheets Trigger to this node.

3. **Add Code node to filter rows:**  
   - Type: `code`  
   - Code: `return items.filter(item => !item.json.Status?.trim());`  
   - Connect output of Read Rows node here.

4. **Add Wait node:**  
   - Type: `wait`  
   - Use default settings to throttle processing.  
   - Connect output of filter code node.

5. **Add Code node for Broad Words:**  
   - Type: `code`  
   - Code to extract last three words of `Blog Inspiration` column or fallback to "photo ideas":  
   ```js
   return items.map(item => {
     const inspiration = item.json["Blog Inspiration"];
     const words = inspiration.split(" ");
     const shortTopic = words.slice(-3).join(" ");
     item.json.topic = shortTopic || "photo ideas";
     return item;
   });
   ```  
   - Connect output of Wait node here.

6. **Add HTTP Request node for Autocomplete:**  
   - Type: `httpRequest`  
   - Method: POST  
   - URL: `https://seo-api2.onrender.com/get-seo-data`  
   - Body: JSON with parameter `topic` set to `{{$json.topic}}`  
   - Set max retries to 5, retry delay 5000 ms.  
   - Connect output of Broad Words node.

7. **Add HTTP Request node for SerpAPI PAA:**  
   - Type: `httpRequest`  
   - Method: GET  
   - URL: `https://serpapi.com/search`  
   - Query parameters:  
     - `q`: `={{ $('Only Reads Empty Status').item.json['Blog Inspiration'] }}`  
     - `api_key`: Your SerpAPI key (credential)  
     - `engine`: `google`  
     - `hl`: `en`  
     - `google_domain`: `google.com`  
     - `device`: `desktop`  
   - Connect output of Wait node here (parallel to Autocomplete).

8. **Add Code node to format SerpAPI PAA results:**  
   - Type: `code`  
   - Use provided code to extract `related_questions`, `organic_results`, `answer_box`, and `related_searches` with fallbacks, formatting as string array.  
   - Connect output of PAA (SerpAPI) node.

9. **Add Merge node:**  
   - Merge inputs from Autocomplete and Format PAA nodes by index.

10. **Add Code node "All of the Information":**  
    - Type: `code`  
    - Merge Autocomplete, SerpAPI PAA, and broad topic data by normalized topic strings.  
    - Output objects with `topic`, `autocomplete` array, and merged `paa` array (or undefined if empty).  
    - Connect output of Merge node.

11. **Add SplitInBatches node "Loop Over Items":**  
    - Type: `splitInBatches`  
    - Default batch size = 1 for sequential processing.  
    - Connect output of "All of the Information" node.

12. **Add LangChain Agent node "Generate Blog Post":**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Prompt: Use detailed prompt instructing GPT-4 to write warm, SEO-optimized blog posts referencing autocomplete keywords and PAA questions naturally, approx. 500 words, with call-to-action.  
    - Connect output of Loop Over Items.

13. **Add LangChain LM Chat OpenAI node "GPT-4":**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: `gpt-4o`  
    - Connect output of Generate Blog Post.

14. **Add Set node "Update 'Done' Status":**  
    - Type: `set`  
    - Assignments:  
      - `Blog Draft ` = `={{ $json.output }}` (generated blog text)  
      - `Blog Inspiration` = `={{ $('Loop Over Items').item.json.topic }}`  
      - `Status` = `"done"`  
    - Connect output of Generate Blog Post.

15. **Add Google Sheets node "Export":**  
    - Type: `googleSheets`  
    - Operation: Update  
    - Configure with OAuth2 credentials; specify document and sheet.  
    - Map updated fields to corresponding columns.  
    - Connect output of Update "Done" Status.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed to generate SEO-optimized blog posts using Google Autocomplete and PAA data to enrich content ideas, then leveraging GPT-4 for natural, human-like writing. It is ideal for businesses or SEO professionals seeking to automate content creation based on dynamic keyword research.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Project Description (Sticky Note5)                                                              |
| The custom SEO API at https://seo-api2.onrender.com/get-seo-data is a public web service hosted on Render, which extracts Google Autocomplete and PAA data. It may have latency if unused for 15+ minutes due to free tier sleep.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1 Explanation                                                                        |
| SerpAPI is used to fetch additional PAA data leveraging a paid API key. Documentation and API key management should be handled externally.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | SerpAPI HTTP Request node usage                                                                 |
| OpenAI GPT-4 model is used via LangChain integration for blog post generation. Be aware of API quotas, response times, and costs. OpenAI credentials must be configured properly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | GPT-4 Node configuration                                                                        |
| Google Sheets must have at least three columns: "Blog Inspiration" (input topics), "Status" (used for filtering and updated to "done"), and "Blog Draft" (output blog text).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Google Sheets Setup Instructions                                                               |
| For further help on LangChain and SerpAPI integration within n8n, refer to the official n8n documentation: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolserpapi/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Official n8n Docs                                                                               |
| When triggering manually or via webhook, you can replace the Google Sheets Trigger node with other trigger types such as Cron or Webhook to suit automation needs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Workflow flexibility note                                                                       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---