Daily Newsletter Service using Excel, Outlook and AI

https://n8nworkflows.xyz/workflows/daily-newsletter-service-using-excel--outlook-and-ai-3446


# Daily Newsletter Service using Excel, Outlook and AI

### 1. Workflow Overview

This workflow implements a **Daily Newsletter Service** that automatically delivers a curated digest of the latest n8n.io templates to subscribers based on their category preferences. It is designed to run once daily and sends personalized emails summarizing new templates relevant to each subscriber.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled trigger initiates the workflow and reads subscriber data (name, email, categories) from an Excel workbook.
- **1.2 Category Extraction and Template Fetching:** Extracts unique categories from all subscribers and fetches the latest 10 templates per category from the n8n.io API.
- **1.3 AI Summarization:** Uses OpenAI GPT-4o-mini model to generate concise summaries of each fetched template’s description.
- **1.4 Subscriber-specific Filtering:** For each subscriber, filters templates to only those matching their categories, removes duplicates and previously seen templates.
- **1.5 Email Generation and Delivery:** Generates a personalized HTML newsletter and sends it via Microsoft Outlook.

This modular design optimizes API calls by fetching templates once per unique category and then tailoring the digest per subscriber, ensuring efficiency and scalability.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow daily at 6 AM and reads subscriber information from an Excel worksheet. It parses subscriber names, emails, and category preferences for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Subscribers (Microsoft Excel)  
  - Parse Rows (Set)  
  - Get Unique Categories (Set)  
  - Merge (Merge)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Configuration: Runs daily at 6:00 AM  
    - Input: None (trigger)  
    - Output: Initiates workflow  
    - Edge cases: Misconfigured schedule or timezone issues could delay execution.

  - **Get Subscribers**  
    - Type: Microsoft Excel node  
    - Configuration: Reads rows from a specified Excel workbook and worksheet containing subscriber data  
    - Credentials: Microsoft Excel OAuth2  
    - Input: Trigger from Schedule Trigger  
    - Output: Raw subscriber rows  
    - Edge cases: Excel file access errors, invalid OAuth credentials, empty or malformed data.

  - **Parse Rows**  
    - Type: Set node  
    - Configuration: Extracts and formats subscriber fields:  
      - `name` as string  
      - `email` as string  
      - `categories` as array (splits comma-delimited string)  
    - Input: Output from Get Subscribers  
    - Output: Structured subscriber JSON  
    - Edge cases: Missing or malformed category strings could cause split errors.

  - **Get Unique Categories**  
    - Type: Set node (executeOnce=true)  
    - Configuration: Aggregates all subscriber categories into a unique array using expression:  
      `={{ $input.all().flatMap(item => item.json.categories).unique() }}`  
    - Input: Output from Parse Rows (all subscribers)  
    - Output: Unique list of categories  
    - Edge cases: Empty subscriber list results in empty categories array.

  - **Merge**  
    - Type: Merge node (chooseBranch mode)  
    - Configuration: Merges subscriber data with unique categories for downstream processing  
    - Input: From Parse Rows and Get Unique Categories  
    - Output: Combined data stream  
    - Edge cases: Merge conflicts if data streams are out of sync.

---

#### 1.2 Category Extraction and Template Fetching

- **Overview:**  
  This block splits the unique categories into individual items, fetches the latest 10 workflows per category from the n8n.io API, and appends category metadata to each fetched workflow.

- **Nodes Involved:**  
  - Categories to Items (Split Out)  
  - Fetch Latest 10 per Category (HTTP Request)  
  - Append Category (Set)  
  - For Each Category (Split In Batches)  
  - Flatten Workflows (Set)  
  - Aggregate

- **Node Details:**

  - **Categories to Items**  
    - Type: Split Out node  
    - Configuration: Splits the `categories` array into individual category items under field `category`  
    - Input: Unique categories from Get Unique Categories  
    - Output: One item per category  
    - Edge cases: Empty categories array results in no output.

  - **Fetch Latest 10 per Category**  
    - Type: HTTP Request node  
    - Configuration: Calls `https://n8n.io/api/product-api/workflows/search` with query parameters:  
      - `category` = current category  
      - `rows` = 10  
      - `sort` = `createdAt:desc`  
      - `page` = 1  
    - Input: Single category item from Categories to Items  
    - Output: JSON response with workflows for the category  
    - Edge cases: API rate limits, network errors, invalid category names.

  - **Append Category**  
    - Type: Set node  
    - Configuration: Adds the current category string to each fetched workflow item under `category` field  
    - Input: Output from Fetch Latest 10 per Category  
    - Output: Workflows enriched with category metadata  
    - Edge cases: Missing category field if upstream fails.

  - **For Each Category**  
    - Type: Split In Batches node  
    - Configuration: Iterates over each category item to process workflows in batches  
    - Input: Categories to Items output  
    - Output: Processes each category sequentially  
    - Edge cases: Large category lists may increase execution time.

  - **Flatten Workflows**  
    - Type: Set node (executeOnce=true)  
    - Configuration: Flattens all fetched workflows from all categories into a single array under `workflows` field:  
      `={{ $input.all().flatMap(item => item.json.data) }}`  
    - Input: Aggregated workflows from For Each Category  
    - Output: Single array of all workflows  
    - Edge cases: Empty data arrays cause empty workflows list.

  - **Aggregate**  
    - Type: Aggregate node  
    - Configuration: Aggregates all workflow items into one for downstream processing  
    - Input: Output from Collect Fields (next block)  
    - Output: Aggregated workflows  
    - Edge cases: Large data sets may impact performance.

---

#### 1.3 AI Summarization

- **Overview:**  
  This block uses an OpenAI GPT-4o-mini model to generate concise summaries of each workflow’s description, improving readability for the newsletter.

- **Nodes Involved:**  
  - Workflows to Items (Split Out)  
  - Workflow Summarizer (Langchain Chain LLM)  
  - OpenAI Chat Model (Langchain LLM Chat)  
  - Collect Fields (Set)  
  - Aggregate

- **Node Details:**

  - **Workflows to Items**  
    - Type: Split Out node  
    - Configuration: Splits the aggregated workflows array into individual workflow items under `workflow` field  
    - Input: Flatten Workflows output  
    - Output: One workflow per item  
    - Edge cases: Empty workflows array results in no output.

  - **Workflow Summarizer**  
    - Type: Langchain Chain LLM node  
    - Configuration: Uses a prompt to summarize the workflow description into 1-2 sentences, highlighting goal, method, and key nodes.  
      - Input text: Workflow description with markdown characters removed  
      - Prompt example:  
        "You have received a description of a n8n template from the official template gallery. Your task is to summarize the description into one or two sentences..."  
    - Input: Single workflow item  
    - Output: Summarized text in `response.text`  
    - Edge cases: API errors, rate limits, or empty descriptions.

  - **OpenAI Chat Model**  
    - Type: Langchain LLM Chat node  
    - Configuration: Uses GPT-4o-mini model to process the summarization prompt  
    - Credentials: OpenAI API key  
    - Input: Workflow Summarizer node  
    - Output: AI-generated summary  
    - Edge cases: Authentication errors, model unavailability.

  - **Collect Fields**  
    - Type: Set node  
    - Configuration: Collects and maps relevant workflow fields and AI summary into a structured object:  
      - workflow_id, workflow_name, workflow_desc (AI summary), workflow_created_at, author_id, author_name, author_username, category  
    - Input: Workflow Summarizer output and original workflow data  
    - Output: Structured workflow summary object  
    - Edge cases: Missing fields in source data.

  - **Aggregate**  
    - Type: Aggregate node  
    - Configuration: Aggregates all summarized workflow items into one array for filtering  
    - Input: Collect Fields output  
    - Output: Aggregated summarized workflows  
    - Edge cases: Large data sets may impact performance.

---

#### 1.4 Subscriber-specific Filtering

- **Overview:**  
  For each subscriber, this block filters the summarized workflows to only those matching their subscribed categories, removes duplicates and workflows already sent in previous executions.

- **Nodes Involved:**  
  - For Each Subscriber (Split In Batches)  
  - With User Reference (Set)  
  - Get Relevant Workflows (Set)  
  - Workflow to Items (Split Out)  
  - Remove Already Seen (Remove Duplicates)  
  - Combine Workflows (Aggregate)  
  - Has New Workflows? (If)  
  - No Operation, do nothing (NoOp)

- **Node Details:**

  - **For Each Subscriber**  
    - Type: Split In Batches node  
    - Configuration: Iterates over each subscriber to process their personalized digest  
    - Input: Subscriber list with categories and aggregated workflows  
    - Output: One subscriber per batch  
    - Edge cases: Large subscriber lists increase execution time.

  - **With User Reference**  
    - Type: Set node  
    - Configuration: Adds subscriber fields (`name`, `email`, `categories`) to the data stream for reference  
    - Input: For Each Subscriber output  
    - Output: Subscriber data enriched for filtering  
    - Edge cases: Missing subscriber fields.

  - **Get Relevant Workflows**  
    - Type: Set node  
    - Configuration: Filters the aggregated workflows to only those matching the subscriber’s categories:  
      `={{ $json.categories.flatMap(cat => $('Flatten Workflows').first().json.workflows.filter(item => item.category === cat)) }}`  
    - Input: Subscriber categories and flattened workflows  
    - Output: Workflows relevant to the subscriber  
    - Edge cases: No matching workflows results in empty array.

  - **Workflow to Items**  
    - Type: Split Out node  
    - Configuration: Splits relevant workflows array into individual items for deduplication  
    - Input: Get Relevant Workflows output  
    - Output: Individual workflow items  
    - Edge cases: Empty workflows array.

  - **Remove Already Seen**  
    - Type: Remove Duplicates node  
    - Configuration: Removes workflows already sent to the subscriber in previous executions using a dedupe key:  
      `={{ $('For Each Subscriber').item.json.name.toSnakeCase() }}_{{ $json.workflow_id }}`  
    - Input: Workflow to Items output  
    - Output: New workflows only  
    - Edge cases: Persistence issues may cause duplicates.

  - **Combine Workflows**  
    - Type: Aggregate node  
    - Configuration: Aggregates new workflows back into an array for email generation  
    - Input: Remove Already Seen output  
    - Output: Array of new workflows  
    - Edge cases: Empty input results in empty array.

  - **Has New Workflows?**  
    - Type: If node  
    - Configuration: Checks if there are any new workflows to send (`length > 0`)  
    - Input: Combine Workflows output  
    - Output: Branches to send email or no operation  
    - Edge cases: False positives if data malformed.

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Configuration: Placeholder for subscribers with no new workflows  
    - Input: Has New Workflows? false branch  
    - Output: None  
    - Edge cases: None.

---

#### 1.5 Email Generation and Delivery

- **Overview:**  
  This block generates a personalized HTML newsletter for each subscriber with new workflows and sends it via Microsoft Outlook.

- **Nodes Involved:**  
  - Generate HTML Template (HTML)  
  - Send Daily Digest (Microsoft Outlook)

- **Node Details:**

  - **Generate HTML Template**  
    - Type: HTML node  
    - Configuration: Uses a template to generate an HTML email body including:  
      - Date header  
      - Sections per category with workflow links, authors, creation dates, and AI summaries  
    - Input: New workflows array and subscriber categories  
    - Output: HTML string for email body  
    - Edge cases: Empty workflows cause empty email body.

  - **Send Daily Digest**  
    - Type: Microsoft Outlook node  
    - Configuration: Sends email with:  
      - Subject: "New Workflows for [Day]"  
      - Body: HTML content from previous node  
      - To: Subscriber email  
      - From and Reply-To: no-reply@example.com  
    - Credentials: Microsoft Outlook OAuth2  
    - Input: HTML email content and subscriber email  
    - Output: Email sent confirmation  
    - Edge cases: Authentication errors, email delivery failures, invalid email addresses.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                               | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                              |
|---------------------------|---------------------------------|-----------------------------------------------|---------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                 | Initiates workflow daily at 6 AM               | None                      | Get Subscribers             |                                                                                                        |
| Get Subscribers           | Microsoft Excel                 | Reads subscriber list from Excel workbook      | Schedule Trigger          | Parse Rows                  | ## 1. Get Subscribers from Excel [Learn more about the Excel node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel) Excel can be an easy way to store a simple list of subscribers who will receive our daily digest. We can also specify only the categories they are interested in. |
| Parse Rows                | Set                            | Parses and formats subscriber fields           | Get Subscribers           | Get Unique Categories, Merge | ### Columns - name *(text)* - email *(text)* - categories *(text, comma-delimited)*                     |
| Get Unique Categories     | Set                            | Extracts unique categories from all subscribers| Parse Rows                | Categories to Items          |                                                                                                        |
| Merge                    | Merge                          | Merges subscriber data and unique categories   | Parse Rows, Get Unique Categories | For Each Category          |                                                                                                        |
| Categories to Items       | Split Out                      | Splits unique categories into individual items | Get Unique Categories     | Fetch Latest 10 per Category | ## 2. Fetch Latest Templates from n8n [Learn more about the HTTP Request node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) Using the HTTP request node, we can call the n8n.io template search API to the latest published templates. However, to save on resources, we only want to fetch from categories relevant to our subscribers. To do so: 1) We only want to fetch latest templates from unique categories amongst all subscribers 2) Do this fetching once to later reference for all subscribers |
| Fetch Latest 10 per Category | HTTP Request                  | Fetches latest 10 workflows per category       | Categories to Items       | Append Category             |                                                                                                        |
| Append Category           | Set                            | Adds category metadata to each fetched workflow| Fetch Latest 10 per Category | For Each Category          |                                                                                                        |
| For Each Category         | Split In Batches               | Iterates over each category to fetch workflows | Merge                     | Flatten Workflows, Workflows to Items |                                                                                                        |
| Flatten Workflows         | Set                            | Flattens all workflows into a single array     | For Each Category         | Merge                       | ### Execute Once This node has been set to execute once rather than for each subscriber.                 |
| Workflows to Items        | Split Out                      | Splits workflows array into individual items   | Flatten Workflows         | Workflow Summarizer         |                                                                                                        |
| Workflow Summarizer       | Langchain Chain LLM            | Summarizes workflow descriptions using AI      | Workflows to Items        | Collect Fields              | ## 3. Generate AI Summary For Each Template [Read more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) When building our email digest, we'd rather have a shortened and summarized version of each template's description for easier scanning and reading. We can use AI to accomplish this and merge it with the template object. |
| OpenAI Chat Model         | Langchain LLM Chat             | Runs GPT-4o-mini model for summarization       | Workflow Summarizer       | Workflow Summarizer (ai_languageModel) |                                                                                                        |
| Collect Fields            | Set                            | Collects and maps workflow and summary fields  | Workflow Summarizer       | Aggregate                   |                                                                                                        |
| Aggregate                 | Aggregate                      | Aggregates all summarized workflows             | Collect Fields            | For Each Subscriber         |                                                                                                        |
| For Each Subscriber       | Split In Batches               | Iterates over each subscriber                    | Aggregate                 | Has New Workflows?, Get Relevant Workflows |                                                                                                        |
| With User Reference       | Set                            | Adds subscriber info for filtering               | Combine Workflows         | For Each Subscriber         |                                                                                                        |
| Get Relevant Workflows    | Set                            | Filters workflows by subscriber categories       | For Each Subscriber       | Workflow to Items           | ## 4. Filter Relevant Templates for Subscriber [Read more about the Split Out node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitout) For each subscriber, we want to filter out our freshly collected n8n.io templates by the categories relevant to the subscriber as defined in the Excel sheet. A "Remove duplicates" node can be used to keep track of duplicate templates - as templates can have more than one category and appear twice! |
| Workflow to Items         | Split Out                      | Splits relevant workflows into individual items | Get Relevant Workflows    | Remove Already Seen         |                                                                                                        |
| Remove Already Seen       | Remove Duplicates              | Removes workflows already sent to subscriber    | Workflow to Items         | Combine Workflows           |                                                                                                        |
| Combine Workflows         | Aggregate                      | Aggregates new workflows for email generation   | Remove Already Seen       | With User Reference         |                                                                                                        |
| Has New Workflows?        | If                             | Checks if subscriber has new workflows to send  | For Each Subscriber       | Generate HTML Template, No Operation, do nothing |                                                                                                        |
| No Operation, do nothing  | NoOp                           | Placeholder for no new workflows                 | Has New Workflows? (false) | None                       |                                                                                                        |
| Generate HTML Template    | HTML                           | Generates personalized HTML newsletter           | Has New Workflows? (true) | Send Daily Digest           | ## 5. Generate Daily Digest and Send Via Outlook [Read more about the Outlook node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook) Finally, we can construct our digest's content using the HTML node and customise it by subscriber as necessary. The Outlook node is then used to send the digest to the subscriber. |
| Send Daily Digest         | Microsoft Outlook              | Sends the newsletter email to subscriber         | Generate HTML Template    | None                       |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run daily at 6:00 AM.

2. **Add Microsoft Excel node ("Get Subscribers")**  
   - Operation: Read Rows from worksheet  
   - Configure workbook and worksheet IDs for your subscriber Excel file.  
   - Connect Schedule Trigger → Get Subscribers.  
   - Set Microsoft Excel OAuth2 credentials.

3. **Add Set node ("Parse Rows")**  
   - Extract fields:  
     - `name` = `{{$json.name}}`  
     - `email` = `{{$json.email}}`  
     - `categories` = `{{$json.categories.split(',')}}` (convert comma-delimited string to array)  
   - Connect Get Subscribers → Parse Rows.

4. **Add Set node ("Get Unique Categories")**  
   - Set `categories` field with expression:  
     `={{ $input.all().flatMap(item => item.json.categories).unique() }}`  
   - Enable "Execute Once" option.  
   - Connect Parse Rows → Get Unique Categories.

5. **Add Merge node ("Merge")**  
   - Mode: Choose Branch  
   - Connect Parse Rows → Merge (first input)  
   - Connect Get Unique Categories → Merge (second input).

6. **Add Split Out node ("Categories to Items")**  
   - Field to split out: `categories`  
   - Destination field name: `category`  
   - Connect Get Unique Categories → Categories to Items.

7. **Add HTTP Request node ("Fetch Latest 10 per Category")**  
   - Method: GET  
   - URL: `https://n8n.io/api/product-api/workflows/search`  
   - Query parameters:  
     - `category` = `={{$json.category}}`  
     - `rows` = 10  
     - `sort` = `createdAt:desc`  
     - `page` = 1  
   - Connect Categories to Items → Fetch Latest 10 per Category.

8. **Add Set node ("Append Category")**  
   - Add field `category` = `={{ $('Categories to Items').item.json.category }}`  
   - Include other fields from input.  
   - Connect Fetch Latest 10 per Category → Append Category.

9. **Add Split In Batches node ("For Each Category")**  
   - Connect Merge → For Each Category.

10. **Connect For Each Category → Append Category**  
    - Connect Append Category → For Each Category (batch output).

11. **Add Set node ("Flatten Workflows")**  
    - Set field `workflows` = `={{ $input.all().flatMap(item => item.json.data) }}`  
    - Enable "Execute Once".  
    - Connect For Each Category → Flatten Workflows.

12. **Add Split Out node ("Workflows to Items")**  
    - Field to split out: `workflows`  
    - Destination field name: `workflow`  
    - Connect Flatten Workflows → Workflows to Items.

13. **Add Langchain Chain LLM node ("Workflow Summarizer")**  
    - Input text:  
      ```
      =## Description
      ```
      {{ $json.workflow.description.replaceAll('#', '') }}
      ```
      ```  
    - Prompt: Summarize description into 1-2 sentences highlighting goal, method, and key nodes.  
    - Connect Workflows to Items → Workflow Summarizer.

14. **Add Langchain LLM Chat node ("OpenAI Chat Model")**  
    - Model: GPT-4o-mini  
    - Credentials: OpenAI API key  
    - Connect Workflow Summarizer → OpenAI Chat Model (ai_languageModel input).

15. **Add Set node ("Collect Fields")**  
    - Map fields:  
      - `workflow_id` = `={{ $('Workflows to Items').item.json.workflow.id }}`  
      - `workflow_name` = `={{ $('Workflows to Items').item.json.workflow.name }}`  
      - `workflow_desc` = `={{ $json.response.text }}` (AI summary)  
      - `workflow_created_at` = `={{ $('Workflows to Items').item.json.workflow.createdAt }}`  
      - `author_id` = `={{ $('Workflows to Items').item.json.workflow.user.id }}`  
      - `author_name` = `={{ $('Workflows to Items').item.json.workflow.user.name }}`  
      - `author_username` = `={{ $('Workflows to Items').item.json.workflow.user.username }}`  
      - `category` = `={{ $('For Each Category').item.json.category }}`  
    - Connect Workflow Summarizer → Collect Fields.

16. **Add Aggregate node ("Aggregate")**  
    - Aggregate all items into one array.  
    - Connect Collect Fields → Aggregate.

17. **Add Split In Batches node ("For Each Subscriber")**  
    - Connect Aggregate → For Each Subscriber.

18. **Add Set node ("With User Reference")**  
    - Add subscriber fields:  
      - `name` = `={{ $('For Each Subscriber').item.json.name }}`  
      - `email` = `={{ $('For Each Subscriber').item.json.email }}`  
      - `categories` = `={{ $('For Each Subscriber').item.json.categories }}`  
    - Connect Combine Workflows → With User Reference.

19. **Add Set node ("Get Relevant Workflows")**  
    - Filter workflows by subscriber categories:  
      ```
      ={
        workflows: $json.categories.flatMap(cat =>
          $('Flatten Workflows').first().json.workflows.filter(item => item.category === cat)
        )
      }
      ```  
    - Connect For Each Subscriber → Get Relevant Workflows.

20. **Add Split Out node ("Workflow to Items")**  
    - Field to split out: `workflows`  
    - Connect Get Relevant Workflows → Workflow to Items.

21. **Add Remove Duplicates node ("Remove Already Seen")**  
    - Operation: Remove items seen in previous executions  
    - Dedupe value:  
      `={{ $('For Each Subscriber').item.json.name.toSnakeCase() }}_{{ $json.workflow_id }}`  
    - Connect Workflow to Items → Remove Already Seen.

22. **Add Aggregate node ("Combine Workflows")**  
    - Aggregate all new workflows into array  
    - Connect Remove Already Seen → Combine Workflows.

23. **Add If node ("Has New Workflows?")**  
    - Condition: Check if `length($json.data) > 0`  
    - Connect For Each Subscriber → Has New Workflows?.

24. **Add HTML node ("Generate HTML Template")**  
    - HTML template includes date, categories, workflow links, authors, creation dates, and AI summaries.  
    - Connect Has New Workflows? (true) → Generate HTML Template.

25. **Add Microsoft Outlook node ("Send Daily Digest")**  
    - Subject: `New Workflows for {{ $now.format('DD') }}`  
    - Body: HTML content from previous node  
    - To: Subscriber email from Has New Workflows? node  
    - From and Reply-To: no-reply@example.com  
    - Credentials: Microsoft Outlook OAuth2  
    - Connect Generate HTML Template → Send Daily Digest.

26. **Add No Operation node ("No Operation, do nothing")**  
    - Connect Has New Workflows? (false) → No Operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow builds a daily digest newsletter service that pulls and summarizes the latest n8n.io templates based on subscriber preferences. It is scheduled to run once a day and sends personalized emails.                                                                                                                                                 | Workflow description and purpose                                                                    |
| Excel can be an easy way to store a simple list of subscribers who will receive our daily digest. We can also specify only the categories they are interested in.                                                                                                                                                                                              | [Microsoft Excel node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel) |
| Using the HTTP request node, we can call the n8n.io template search API to get the latest published templates. To save resources, we only fetch templates from unique categories relevant to subscribers.                                                                                                                                                     | [HTTP Request node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| When building our email digest, we'd rather have a shortened and summarized version of each template's description for easier scanning and reading. We use AI to accomplish this and merge it with the template object.                                                                                                                                         | [Basic LLM node documentation](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| For each subscriber, we filter out templates by their categories and remove duplicates to avoid sending repeated content.                                                                                                                                                                                                                                    | [Split Out node documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.splitout) |
| Finally, we construct the digest content using the HTML node and send it via Outlook.                                                                                                                                                                                                                                                                          | [Outlook node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook) |
| To subscribe a new user, add their email and categories to the Excel sheet; to unsubscribe, remove them. Categories must match n8n template website categories (e.g., AI, SecOps, Sales, etc.).                                                                                                                                                                | Usage instructions                                                                                  |
| Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/) for help.                                                                                                                                                                                                                                          | Community support links                                                                             |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and modify the Daily Newsletter Service workflow, anticipate potential issues, and integrate it into their automation environment effectively.