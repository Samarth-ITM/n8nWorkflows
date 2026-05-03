Categorize SEO Keywords & Create Content Strategy with AI and Airtable

https://n8nworkflows.xyz/workflows/categorize-seo-keywords---create-content-strategy-with-ai-and-airtable-8721


# Categorize SEO Keywords & Create Content Strategy with AI and Airtable

### 1. Workflow Overview

This workflow automates the process of categorizing SEO keywords, clustering them into thematic groups, and generating content strategy ideas using AI, leveraging Airtable as the data repository. It is designed for SEO professionals and content strategists aiming to organize large keyword datasets efficiently and produce targeted content plans.

The workflow is logically divided into the following blocks:

- **1.1 Setup & Input Reception:** Initialization and loading of keyword data from Airtable.
- **1.2 Keyword Categorization:** AI-driven categorization of each keyword into SEO-relevant classes.
- **1.3 Keyword Clustering:** Semantic clustering of keywords into coherent topic groups using AI.
- **1.4 Content Idea Generation:** Creation of SEO-optimized titles and descriptions for content opportunities based on categorized and clustered keywords.
- **1.5 Data Storage in Airtable:** Writing categorized keywords, clusters, and content ideas back into respective Airtable tables.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Input Reception

**Overview:**  
This block initializes the workflow by setting Airtable base and table IDs, then retrieves all keywords from the Master Keyword Variations table.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Airtable Fields  
- Airtable Get All KWs  
- Set WF Fields

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts workflow execution manually.  
  - *Connections:* Outputs to "Set Airtable Fields".  
  - *Failures:* None (manual trigger).

- **Set Airtable Fields**  
  - *Type:* Set  
  - *Role:* Stores Airtable base ID and table IDs for later use.  
  - *Configuration:* Hardcoded strings for `airtable_base_id` and IDs for multiple tables (categories, content ideas, clusters, master keywords).  
  - *Inputs/Outputs:* Receives manual trigger input; outputs configuration data to "Airtable Get All KWs".  
  - *Failures:* Misconfigured or outdated IDs will cause downstream Airtable node failures.

- **Airtable Get All KWs**  
  - *Type:* Airtable (Search operation)  
  - *Role:* Fetches all keyword records from the Master KW Variations table.  
  - *Configuration:* Uses base and table IDs from prior node; requires Airtable Personal Access Token credentials.  
  - *Inputs/Outputs:* Inputs from "Set Airtable Fields"; outputs full keyword dataset to "Set WF Fields".  
  - *Failures:* API rate limits, credential errors, or invalid IDs.

- **Set WF Fields**  
  - *Type:* Set  
  - *Role:* Maps incoming Airtable keyword fields into custom workflow fields for convenience (e.g., keyword, competition, MSV).  
  - *Configuration:* Extracts relevant fields from Airtable JSON response for later processing.  
  - *Inputs/Outputs:* From "Airtable Get All KWs"; outputs to "Aggregate Keywords for Agent" and "Category AI Agent".  
  - *Failures:* Expression errors if expected fields are missing.

---

#### 2.2 Keyword Categorization

**Overview:**  
Categorizes each keyword into predefined SEO categories (Quick Wins, Authority Builders, Emerging Topics, Intent Signals, Semantic Topics, Unknown) using an AI agent based on keyword metrics and search intent.

**Nodes Involved:**  
- Aggregate Keywords for Agent  
- Set Field for Agent  
- Category AI Agent  
- Set Category Table Fields  
- Category Table  
- Filter Out Unknown  
- Set Content from Category Fields  
- Loop Over Items  
- Content Ideas from Category AI Agent  
- Set Content Ideas Table Field  
- Categories Content Ideas Table

**Node Details:**

- **Aggregate Keywords for Agent**  
  - *Type:* Aggregate  
  - *Role:* Aggregates keyword data fields (`keyword`, `primary_keyword`, `type`, `search_intent`) into a single dataset string for AI input.  
  - *Inputs/Outputs:* From "Set WF Fields"; outputs aggregated data to "Set Field for Agent".  
  - *Failures:* Incorrect field names or empty input.

- **Set Field for Agent**  
  - *Type:* Set  
  - *Role:* Prepares a JSON string named `keyword_dataset` for AI consumption.  
  - *Inputs/Outputs:* From "Aggregate Keywords for Agent"; outputs to "AI Agent Analyze and Cluster KWs".  
  - *Failures:* Expression errors.

- **Category AI Agent**  
  - *Type:* Langchain Agent (AI)  
  - *Role:* Categorizes individual keywords based on rules involving MSV, difficulty, competition, and intent signals.  
  - *Configuration:* Uses a system prompt defining precise categorization rules and outputs a strict JSON object per keyword.  
  - *Inputs/Outputs:* Receives keywords one by one; outputs categorized keyword JSON to "Set Category Table Fields".  
  - *Failures:* AI model timeout, rate limits, misinterpretation of rules.

- **Set Category Table Fields**  
  - *Type:* Set  
  - *Role:* Parses AI JSON output and maps fields (`keyword`, `category`, `reasoning`, etc.) for Airtable.  
  - *Inputs/Outputs:* From "Category AI Agent"; outputs to "Category Table".  
  - *Failures:* JSON parsing errors if AI output is malformed.

- **Category Table**  
  - *Type:* Airtable (Create operation)  
  - *Role:* Writes categorized keyword records into the Keyword Categories Airtable table.  
  - *Credentials:* Airtable Personal Access Token.  
  - *Inputs/Outputs:* From "Set Category Table Fields"; outputs to "Filter Out Unknown".  
  - *Failures:* API errors, invalid field mapping.

- **Filter Out Unknown**  
  - *Type:* Filter  
  - *Role:* Removes keywords categorized as "Unknown" from further processing.  
  - *Inputs/Outputs:* From "Category Table"; outputs to "Set Content from Category Fields".  
  - *Failures:* Logic errors, empty input sets.

- **Set Content from Category Fields**  
  - *Type:* Set  
  - *Role:* Prepares content-related fields (`keyword`, `category`, `reasoning`) for content idea generation.  
  - *Inputs/Outputs:* From "Filter Out Unknown"; outputs to "Loop Over Items".  
  - *Failures:* Missing field errors.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each keyword record for content idea generation.  
  - *Inputs/Outputs:* From "Set Content from Category Fields"; outputs to "Content Ideas from Category AI Agent".  
  - *Failures:* Batch size misconfiguration.

- **Content Ideas from Category AI Agent**  
  - *Type:* Langchain Agent (AI)  
  - *Role:* Generates SEO-optimized titles and descriptions for each categorized keyword.  
  - *Configuration:* System prompt instructs AI to produce engaging titles and detailed descriptions aligning with keyword category and reasoning.  
  - *Inputs/Outputs:* Takes single keyword input; outputs JSON with title and description to "Set Content Ideas Table Field".  
  - *Failures:* AI model latency or errors.

- **Set Content Ideas Table Field**  
  - *Type:* Set  
  - *Role:* Maps AI output fields into structured data for Airtable insertion, including linking back to the primary keyword and category.  
  - *Inputs/Outputs:* From "Content Ideas from Category AI Agent"; outputs to "Categories Content Ideas Table".  
  - *Failures:* Parsing errors.

- **Categories Content Ideas Table**  
  - *Type:* Airtable (Create operation)  
  - *Role:* Stores generated content ideas (title, description) linked to categorized keywords.  
  - *Credentials:* Airtable Personal Access Token.  
  - *Inputs/Outputs:* From "Set Content Ideas Table Field".  
  - *Failures:* API errors, invalid field mapping.

---

#### 2.3 Keyword Clustering

**Overview:**  
Analyzes the entire keyword dataset holistically to identify semantic clusters and group related keywords by topic and user intent using AI.

**Nodes Involved:**  
- Aggregate Keywords for Agent (shared)  
- Set Field for Agent (shared)  
- AI Agent Analyze and Cluster KWs  
- Edit Fields1  
- Split Out  
- Set Fields for Airtable  
- Split Out1  
- Clusters table

**Node Details:**

- **AI Agent Analyze and Cluster KWs**  
  - *Type:* Langchain Agent (AI)  
  - *Role:* Receives the full keyword dataset and returns clustered groups with fields such as cluster name, core topic, intent pattern, keywords array, reasoning, and primary keyword.  
  - *Configuration:* System prompt guides AI to create natural semantic clusters with no predefined count, ensuring each keyword belongs to exactly one cluster. Output is a JSON object with an array of clusters.  
  - *Inputs/Outputs:* From "Set Field for Agent"; outputs JSON to "Edit Fields1" and "Edit Fields2".  
  - *Failures:* AI resource limits, malformed JSON output.

- **Edit Fields1**  
  - *Type:* Set  
  - *Role:* Extracts clusters array from AI output for individual processing.  
  - *Inputs/Outputs:* From "AI Agent Analyze and Cluster KWs"; outputs to "Split Out".  
  - *Failures:* Expression errors.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits clusters array into individual cluster items for processing.  
  - *Inputs/Outputs:* From "Edit Fields1"; outputs to "Set Fields for Airtable".  
  - *Failures:* Empty input array.

- **Set Fields for Airtable**  
  - *Type:* Set  
  - *Role:* Maps individual cluster fields (`cluster_name`, `core_topic`, `intent_pattern`, `keywords`, `reasoning`, `primary_keyword`) for Airtable storage.  
  - *Inputs/Outputs:* From "Split Out"; outputs to "Split Out1".  
  - *Failures:* Missing fields or type mismatches.

- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Splits the `keywords` array within each cluster into individual keywords for detailed Airtable entry.  
  - *Inputs/Outputs:* From "Set Fields for Airtable"; outputs to "Clusters table".  
  - *Failures:* Empty keywords array.

- **Clusters table**  
  - *Type:* Airtable (Create operation)  
  - *Role:* Records each cluster with core topic, intent, reasoning, primary keyword, and keyword list in the Airtable Clusters table.  
  - *Credentials:* Airtable Personal Access Token.  
  - *Inputs/Outputs:* From "Split Out1".  
  - *Failures:* API errors, field mapping errors.

---

#### 2.4 Content Idea Generation from Clusters

**Overview:**  
Generates hub and spoke content opportunities by creating SEO-optimized titles and descriptions for each cluster and subtopic using AI.

**Nodes Involved:**  
- Edit Fields2  
- Split Out2  
- Agent Create Content Opps  
- Edit Fields3  
- Split Out3  
- Clusters Ideas Table

**Node Details:**

- **Edit Fields2**  
  - *Type:* Set  
  - *Role:* Extracts cluster array and number of clusters from AI clustering output for further processing.  
  - *Inputs/Outputs:* From "AI Agent Analyze and Cluster KWs"; outputs to "Split Out2".  
  - *Failures:* Expression errors.

- **Split Out2**  
  - *Type:* Split Out  
  - *Role:* Splits clusters array into individual clusters for content opportunity creation.  
  - *Inputs/Outputs:* From "Edit Fields2"; outputs to "Agent Create Content Opps".  
  - *Failures:* Empty input array.

- **Agent Create Content Opps**  
  - *Type:* Langchain Agent  
  - *Role:* For each cluster, creates a hub article and up to five spoke articles with SEO-optimized titles and descriptions, following a hub-and-spoke content strategy.  
  - *Configuration:* Prompt specifies JSON structure for each article, character limits, and content guidelines.  
  - *Inputs/Outputs:* Receives cluster data; outputs JSON array to "Edit Fields3".  
  - *Failures:* AI model limits, output formatting errors.

- **Edit Fields3**  
  - *Type:* Set  
  - *Role:* Parses AI response JSON array for individual article splitting.  
  - *Inputs/Outputs:* From "Agent Create Content Opps"; outputs to "Split Out3".  
  - *Failures:* JSON parsing errors.

- **Split Out3**  
  - *Type:* Split Out  
  - *Role:* Splits article array into individual hub/spoke articles.  
  - *Inputs/Outputs:* From "Edit Fields3"; outputs to "Clusters Ideas Table".  
  - *Failures:* Empty input.

- **Clusters Ideas Table**  
  - *Type:* Airtable (Create operation)  
  - *Role:* Stores generated content ideas for clusters with fields like title, description, type (hub/spoke), status, and reasoning into the Content Ideas from Clusters Airtable table.  
  - *Credentials:* Airtable Personal Access Token.  
  - *Inputs/Outputs:* From "Split Out3".  
  - *Failures:* API errors, field mapping issues.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                                | Input Node(s)                | Output Node(s)                          | Sticky Note                                                                                                                                         |
|------------------------------|----------------------------------|-----------------------------------------------|------------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Workflow manual start                          | -                            | Set Airtable Fields                     |                                                                                                                                                     |
| Set Airtable Fields           | Set                              | Sets Airtable base and table IDs               | When clicking ‘Test workflow’ | Airtable Get All KWs                    | Setup instructions and Airtable base/table ID setup.                                                                                               |
| Airtable Get All KWs          | Airtable                         | Retrieves all keywords from master table       | Set Airtable Fields           | Set WF Fields                          |                                                                                                                                                     |
| Set WF Fields                | Set                              | Maps Airtable keyword fields to workflow fields | Airtable Get All KWs          | Aggregate Keywords for Agent, Category AI Agent |                                                                                                                                                     |
| Aggregate Keywords for Agent  | Aggregate                       | Aggregates keyword data into dataset string    | Set WF Fields                | Set Field for Agent                     |                                                                                                                                                     |
| Set Field for Agent           | Set                              | Prepares JSON string for AI input               | Aggregate Keywords for Agent  | AI Agent Analyze and Cluster KWs       |                                                                                                                                                     |
| AI Agent Analyze and Cluster KWs | Langchain Agent (AI)           | Clusters keywords semantically                  | Set Field for Agent           | Edit Fields1, Edit Fields2              | Clusters keywords by semantic similarity and intent.                                                                                              |
| Edit Fields1                 | Set                              | Extracts clusters array                          | AI Agent Analyze and Cluster KWs | Split Out                            |                                                                                                                                                     |
| Split Out                   | Split Out                       | Splits clusters array for processing            | Edit Fields1                 | Set Fields for Airtable                 |                                                                                                                                                     |
| Set Fields for Airtable       | Set                              | Maps cluster data for Airtable                   | Split Out                   | Split Out1                            |                                                                                                                                                     |
| Split Out1                   | Split Out                       | Splits cluster keywords for Airtable            | Set Fields for Airtable       | Clusters table                        |                                                                                                                                                     |
| Clusters table               | Airtable                         | Stores clusters data in Airtable                  | Split Out1                   | -                                      |                                                                                                                                                     |
| Category AI Agent             | Langchain Agent (AI)             | Categorizes keywords into SEO categories         | Set WF Fields                | Set Category Table Fields              | Categorizes keywords as Quick Wins, Authority Builders, Emerging Topics, Intent Signals, Semantic Topics, or Unknown.                              |
| Set Category Table Fields     | Set                              | Maps categorized keyword data for Airtable      | Category AI Agent            | Category Table                        |                                                                                                                                                     |
| Category Table               | Airtable                         | Stores categorized keywords                       | Set Category Table Fields    | Filter Out Unknown                    |                                                                                                                                                     |
| Filter Out Unknown            | Filter                           | Filters out keywords categorized as Unknown      | Category Table              | Set Content from Category Fields       |                                                                                                                                                     |
| Set Content from Category Fields | Set                            | Prepares keyword/category data for content ideas | Filter Out Unknown           | Loop Over Items                      |                                                                                                                                                     |
| Loop Over Items              | Split In Batches                | Iterates over keywords for content generation    | Set Content from Category Fields | Content Ideas from Category AI Agent |                                                                                                                                                     |
| Content Ideas from Category AI Agent | Langchain Agent (AI)         | Generates SEO titles and descriptions            | Loop Over Items              | Set Content Ideas Table Field          | Creates title and description for each categorized keyword.                                                                                        |
| Set Content Ideas Table Field | Set                              | Maps content idea data for Airtable               | Content Ideas from Category AI Agent | Categories Content Ideas Table       |                                                                                                                                                     |
| Categories Content Ideas Table | Airtable                       | Stores content ideas linked to keywords           | Set Content Ideas Table Field | Loop Over Items                      |                                                                                                                                                     |
| Edit Fields2                 | Set                              | Extracts cluster data from AI output              | AI Agent Analyze and Cluster KWs | Split Out2                         |                                                                                                                                                     |
| Split Out2                   | Split Out                       | Splits clusters for content opportunity creation  | Edit Fields2                 | Agent Create Content Opps             |                                                                                                                                                     |
| Agent Create Content Opps     | Langchain Agent                 | Creates hub and spoke content strategy articles   | Split Out2                   | Edit Fields3                         | Creates Title and Description for each categorized keyword. Sends to Airtable.                                                                     |
| Edit Fields3                 | Set                              | Parses AI content opportunity JSON array          | Agent Create Content Opps     | Split Out3                         |                                                                                                                                                     |
| Split Out3                   | Split Out                       | Splits content ideas array into individual items  | Edit Fields3                 | Clusters Ideas Table                 |                                                                                                                                                     |
| Clusters Ideas Table         | Airtable                         | Stores hub and spoke content ideas for clusters   | Split Out3                   | -                                  |                                                                                                                                                     |
| Sticky Note                  | Sticky Note                    | Overview of keyword categorization purpose         | -                            | -                                  | ## Gets KWs from Master List and Categorizes\nCategorizes keywords as Quick Wins, Authority Builders, Emerging Topics, and Unknown.                |
| Sticky Note1                 | Sticky Note                    | Overview of sending categorized keywords to Airtable | -                            | -                                  | ## Send All Categorized Keywords to Airtable                                                                                                      |
| Sticky Note2                 | Sticky Note                    | Overview of creating titles and descriptions for keywords and sending to Airtable | -                            | -                                  | ## Creates Title and Description for each categorized keyword.\nSends to Airtable                                                                 |
| Sticky Note4                 | Sticky Note                    | Overview of keyword clustering                      | -                            | -                                  | ## Clusters KWs from Master KW All Variations List\nCreates clusters based on semantic similarity and search intent.                              |
| Sticky Note5                 | Sticky Note                    | Overview of adding clusters and keywords to Airtable | -                            | -                                  | ## Adds Cluster and Keywords to Clusters Sheet                                                                                                   |
| Sticky Note6                 | Sticky Note                    | Overview of creating hub and spoke content opportunities | -                            | -                                  | ## Create Hub and Spoke Content Opportunities\nCreates title and description for Hub and Spoke content opportunities.\nPrompted (at the moment) to create 5 supporting articles for each pillar. |
| Sticky Note7                 | Sticky Note                    | General workflow overview                           | -                            | -                                  | ## Categorize and Create Content Opportunities                                                                                                   |
| Sticky Note8                 | Sticky Note                    | General overview of clustering and creating content opportunities | -                            | -                                  | ## Cluster and Create Content Opportunities                                                                                                      |
| Sticky Note9                 | Sticky Note                    | Setup instructions for Airtable base and table IDs | -                            | -                                  | # Setup \n\n## 1. Copy this Airtable base: [KW Research Content Ideation](https://airtable.com/apphzhR0wI16xjJJs/shrsojqqzGpgMJq9y)\n## Important: Copy the base. Please do not ask for access. \n## 2. Set Airtable Base Id\nWith your (copied) Airtable base open (to any table), copy the base id from the url.\n\nThe base id begins with app. For example: https://airtable.com/apphzhR0wI16xjJJs/tblewTSMwBdGQKUuZ/\napphzhR0wI16xjJJs is the base id. Enter this into the Set Airtable Fields node.\n\n## 3. Enter the table id of the following tables into the Set Airtable Fields node.\n- Master All KW Variations\n- Keyword Categories\n- Content Ideas for Keywords\n- Clusters\n- Content Ideas from Clusters\n\nThe table id is after the base id. For example.\nhttps://airtable.com/apphzhR0wI16xjJJs/tblD8sMi6W4EikkN4/viw8DZMvccWGY7YuO?blocks=hide\n\ntblD8sMi6W4EikkN4 is the table id.\n\n## 4. Test your automation\nSelect Test Workflow. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" to start the workflow manually.

2. **Add a Set node** named "Set Airtable Fields":  
   - Add string fields for:  
     - `airtable_base_id` (e.g., "apprrQ0Dv1cJOfMi9")  
     - `categories_table_id` (e.g., "tblD8sMi6W4EikkN4")  
     - `content_ideas_from_kws_table_id` (e.g., "tblRDR7uE4b73ZpRt")  
     - `clusters_table_id` (e.g., "tblDRGVjI1vPuJxvm")  
     - `content_ideas_from_clusters_table_id` (e.g., "tbl7trYCu9sSGdRTJ")  
     - `master_all_kw_variations_table_id` (e.g., "tblHz4bwclrB24afu")  
   - Connect manual trigger output to this node.

3. **Add an Airtable node** named "Airtable Get All KWs":  
   - Operation: Search  
   - Base ID and Table ID taken from "Set Airtable Fields".  
   - Credentials: Airtable Personal Access Token.  
   - Connect from "Set Airtable Fields".

4. **Add a Set node** named "Set WF Fields":  
   - Map fields from Airtable output (e.g., `Keyword` → `keyword`, `MSV` → `msv`, etc.).  
   - Connect from "Airtable Get All KWs".

5. **Add an Aggregate node** named "Aggregate Keywords for Agent":  
   - Aggregate all keyword fields: `keyword`, `primary_keyword`, `type`, `search_intent`.  
   - Connect from "Set WF Fields".

6. **Add a Set node** named "Set Field for Agent":  
   - Create a field `keyword_dataset` as a JSON string from aggregated data.  
   - Connect from "Aggregate Keywords for Agent".

7. **Add a Langchain Agent node** named "AI Agent Analyze and Cluster KWs":  
   - Use OpenAI GPT-4 or equivalent.  
   - System prompt instructs semantic clustering as per detailed instructions.  
   - Input: `keyword_dataset` string.  
   - Connect from "Set Field for Agent".

8. **Add two Set nodes** named "Edit Fields1" and "Edit Fields2":  
   - "Edit Fields1": Extract `clusters` array from AI output.  
   - "Edit Fields2": Extract `clusters` array and `number_of_clusters`.  
   - Connect from "AI Agent Analyze and Cluster KWs".

9. **Add Split Out nodes** named "Split Out" and "Split Out2":  
   - "Split Out": Split clusters array for detailed processing. Connect from "Edit Fields1".  
   - "Split Out2": Split clusters array for content opportunities. Connect from "Edit Fields2".

10. **Add Set node** named "Set Fields for Airtable":  
    - Map each cluster's fields: `cluster_name`, `core_topic`, `intent_pattern`, `keywords`, `reasoning`, `primary_keyword`.  
    - Connect from "Split Out".

11. **Add Split Out node** named "Split Out1":  
    - Split `keywords` array from each cluster.  
    - Connect from "Set Fields for Airtable".

12. **Add Airtable node** named "Clusters table":  
    - Operation: Create records with cluster info.  
    - Connect from "Split Out1".  
    - Use Airtable credentials.

13. **Add Langchain Agent node** named "Category AI Agent":  
    - Use GPT-4 with prompt for keyword categorization rules.  
    - Input: single keywords from "Set WF Fields".  
    - Connect from "Set WF Fields".

14. **Add Set node** named "Set Category Table Fields":  
    - Parse AI categorization JSON and prepare fields for Airtable.  
    - Connect from "Category AI Agent".

15. **Add Airtable node** named "Category Table":  
    - Operation: Create categorized keyword records.  
    - Connect from "Set Category Table Fields".

16. **Add Filter node** named "Filter Out Unknown":  
    - Condition: Exclude records where Category = "Unknown".  
    - Connect from "Category Table".

17. **Add Set node** named "Set Content from Category Fields":  
    - Map relevant fields for content idea generation.  
    - Connect from "Filter Out Unknown".

18. **Add Split In Batches node** named "Loop Over Items":  
    - Iterate over keywords for content idea generation.  
    - Connect from "Set Content from Category Fields".

19. **Add Langchain Agent node** named "Content Ideas from Category AI Agent":  
    - Use GPT-4 to generate SEO titles and descriptions per keyword.  
    - Connect from "Loop Over Items".

20. **Add Set node** named "Set Content Ideas Table Field":  
    - Parse AI content idea JSON and prepare Airtable fields.  
    - Connect from "Content Ideas from Category AI Agent".

21. **Add Airtable node** named "Categories Content Ideas Table":  
    - Operation: Create content idea records.  
    - Connect from "Set Content Ideas Table Field".

22. **Add Langchain Agent node** named "Agent Create Content Opps":  
    - For each cluster, create hub and spoke articles with titles and descriptions.  
    - Connect from "Split Out2".

23. **Add Set node** named "Edit Fields3":  
    - Parse AI output JSON array of content opportunities.  
    - Connect from "Agent Create Content Opps".

24. **Add Split Out node** named "Split Out3":  
    - Split content opportunities into individual articles.  
    - Connect from "Edit Fields3".

25. **Add Airtable node** named "Clusters Ideas Table":  
    - Operation: Create hub and spoke content idea records for clusters.  
    - Connect from "Split Out3".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Setup instructions require copying the Airtable base "KW Research Content Ideation" and configuring base/table IDs correctly in the "Set Airtable Fields" node. Base and table ID extraction instructions included.                          | [Airtable KW Research Content Ideation Base](https://airtable.com/apphzhR0wI16xjJJs/shrsojqqzGpgMJq9y)                          |
| The workflow relies heavily on OpenAI GPT-4 language models accessed via Langchain Agent nodes, requiring valid API credentials and sufficient usage quotas.                                                                             | OpenAI API documentation                                                                                                      |
| Categorization rules are explicitly defined in the "Category AI Agent" node to ensure consistent classification of keywords into SEO categories.                                                                                         | Internal system prompt documentation                                                                                            |
| Semantic clustering is performed holistically on the entire keyword dataset, allowing natural groupings without predefined cluster counts, enhancing topical relevance in content strategy.                                                 | Internal system prompt documentation                                                                                            |
| Hub and spoke content creation follows SEO best practices, limiting title length and including descriptions that guide content creation aligned with user intent and keyword categorization.                                               | Internal system prompt documentation                                                                                            |
| The workflow uses multiple "Split Out" and "Split In Batches" nodes to handle batch processing and iteration over complex nested data structures, ensuring scalability for large keyword datasets.                                           | n8n documentation on Split Out and Split In Batches nodes                                                                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.