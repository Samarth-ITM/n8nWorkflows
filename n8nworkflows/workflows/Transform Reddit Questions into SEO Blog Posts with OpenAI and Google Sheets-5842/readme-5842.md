Transform Reddit Questions into SEO Blog Posts with OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/transform-reddit-questions-into-seo-blog-posts-with-openai-and-google-sheets-5842


# Transform Reddit Questions into SEO Blog Posts with OpenAI and Google Sheets

### 1. Workflow Overview

This workflow automates the process of transforming Reddit questions from the "n8n" subreddit into SEO-optimized blog posts, leveraging OpenAI language models and Google Sheets for data storage and management. It is designed for content creators or marketers aiming to generate blog articles based on real user queries, enhancing SEO relevance by using actual questions asked on Reddit.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Fetch recent Reddit posts and filter them to extract only questions.
- **1.2 Data Storage:** Append filtered questions and their details into a Google Sheet ("Queries") for record-keeping.
- **1.3 Iterative AI Processing:** Loop over each question to generate blog post components using AI agents and OpenAI chat models.
- **1.4 Content Generation and Enhancement:** Create and refine blog post elements — title, slug, introduction, steps, conclusion — using multiple AI prompt stages with memory buffers.
- **1.5 Final Data Aggregation:** Compile and update the blog post data into a dedicated Google Sheet ("Blog") for publication or further processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Filtering  
**Overview:**  
This block fetches the latest 30 posts from the "n8n" subreddit and filters them to retain only those that are questions. This ensures subsequent processing focuses on relevant content for blog post generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Reddit  
- Code

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow manually  
  - *Config:* No parameters; triggers on manual test execution  
  - *Connections:* Outputs to Reddit  
  - *Failure Modes:* None (manual trigger)  

- **Reddit**  
  - *Type:* Reddit node  
  - *Role:* Fetch latest posts from subreddit  
  - *Config:*  
    - Subreddit: "n8n"  
    - Limit: 30 posts  
    - Category: "new" (fetch newest posts)  
    - OAuth2 credential configured  
  - *Connections:* Outputs to Code node  
  - *Failure Modes:* Auth errors, rate limiting, network issues  

- **Code**  
  - *Type:* JavaScript Code node  
  - *Role:* Filter posts to keep only questions  
  - *Config:*  
    - Filters posts based on title containing question words or ending with '?'  
    - Uses a list of common question words to detect questions  
    - Logs detected questions for debugging  
  - *Key Expressions:*  
    - `item.json.title` for each post title  
    - Checks if title ends with '?' or starts/includes question words  
  - *Connections:* Outputs filtered questions to Google Sheets  
  - *Failure Modes:* Malformed data (missing title), expression errors  

---

#### 1.2 Data Storage  
**Overview:**  
Stores the filtered Reddit questions (title and selftext) into a Google Sheet named "Queries" for later reference and processing.

**Nodes Involved:**  
- Google Sheets (named "Google Sheets")  

**Node Details:**  

- **Google Sheets**  
  - *Type:* Google Sheets Append node  
  - *Role:* Append new rows with question data  
  - *Config:*  
    - Document ID: VOC to blog Google Sheet  
    - Sheet Name: Queries (gid=0)  
    - Columns: "Query" set to Reddit post body (`selftext`), "Title" set to post title  
    - Mapping mode: Explicit column definition  
    - OAuth2 credentials configured  
  - *Connections:* Outputs to Loop Over Items  
  - *Failure Modes:* Auth errors, quota limits, sheet access issues  

---

#### 1.3 Iterative AI Processing  
**Overview:**  
Processes each stored question individually. Uses a batch loop to handle each question sequentially and passes it through an AI agent that rephrases the question to ensure clarity and consistency.

**Nodes Involved:**  
- Loop Over Items  
- AI Agent  
- Simple Memory (buffer for AI context)  
- OpenAI Chat Model  

**Node Details:**  

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes question items one by one  
  - *Config:* Default settings (batch size unspecified, defaults to 1)  
  - *Connections:* Main output to AI Agent, secondary output unused  
  - *Failure Modes:* None specific, but large batch sizes could cause timeouts  

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Rephrase the question while retaining meaning  
  - *Config:*  
    - Prompt template: "Here is a question: {{ $json.Title }} Rephrase it without changing the meaning. Keep it as a question."  
    - Output: Only the rephrased question  
  - *Connections:* Outputs to "Edit Fields4" (set node)  
  - *Failure Modes:* AI API errors, prompt failures, malformed input data  

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains AI session context using question title as session key  
  - *Config:* Session key derived from question title (`{{ $json.Title }}`)  
  - *Connections:* Feeds into AI Agent's memory input  
  - *Failure Modes:* Memory overflow, session key collisions  

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides the language model for AI Agent processing  
  - *Config:* Uses GPT-4o-mini model  
  - *Connections:* Connected as AI language model input to AI Agent  
  - *Failure Modes:* API rate limits, auth errors, model unavailability  

---

#### 1.4 Content Generation and Enhancement  
**Overview:**  
Transforms the rephrased question into a blog post by generating a title ("name"), a slug for the URL, and blog content sections: Introduction, Steps (a detailed guide), and Conclusion. Each section is produced by dedicated AI agents with memory buffers to maintain context.

**Nodes Involved:**  
- Edit Fields4 (set "name")  
- Google Sheets1 (append/update blog metadata)  
- Intro (AI Agent) + Simple Memory2 + OpenAI Chat Model2 + Edit Fields1  
- Steps (AI Agent) + Simple Memory3 + OpenAI Chat Model3 + Edit Fields2  
- Conclusion (AI Agent) + Simple Memory4 + OpenAI Chat Model4 + Edit Fields3  
- slug (AI Agent) + Simple Memory1 + OpenAI Chat Model1 + Edit Fields  
- Google Sheets2 (append final blog post data)  

**Node Details:**  

- **Edit Fields4**  
  - *Type:* Set node  
  - *Role:* Assigns "name" field with output from rephrased question  
  - *Config:* Sets `name = {{ $json.output }}` (from AI Agent)  
  - *Connections:* To Google Sheets1  
  - *Failure Modes:* None significant  

- **Google Sheets1**  
  - *Type:* Google Sheets appendOrUpdate  
  - *Role:* Store or update blog metadata (name, slug, intro, steps, conclusion) in "Blog" sheet (gid=1732850028)  
  - *Config:*  
    - Document ID: Same as previous  
    - Matching column: "name" (to update existing rows or append new)  
    - OAuth2 credentials set  
  - *Connections:* Outputs to AI agents for content generation (Intro, Steps, Conclusion, slug)  
  - *Failure Modes:* Auth errors, write conflicts, quota  

- **Intro**  
  - *Type:* Langchain Agent  
  - *Role:* Generate short blog post introduction based on "name"  
  - *Config:* Prompt requests an easy-to-read intro with simple vocabulary  
  - *Connections:* Outputs to Edit Fields1  
  - *Failure Modes:* AI errors, prompt issues  

- **Edit Fields1**  
  - *Type:* Set node  
  - *Role:* Assigns "Intro" field with AI output  
  - *Connections:* Outputs to Google Sheets2 for final append  
  - *Failure Modes:* None significant  

- **Steps**  
  - *Type:* Langchain Agent  
  - *Role:* Generate detailed step-by-step guide section for the blog post  
  - *Config:* Prompt requests detailed but easy-to-read steps  
  - *Connections:* Outputs to Edit Fields2  
  - *Failure Modes:* AI errors  

- **Edit Fields2**  
  - *Type:* Set node  
  - *Role:* Assigns "Steps" field with AI output  
  - *Connections:* Outputs to Google Sheets2  
  - *Failure Modes:* None significant  

- **Conclusion**  
  - *Type:* Langchain Agent  
  - *Role:* Generate short blog conclusion with simple vocabulary  
  - *Connections:* Outputs to Edit Fields3  
  - *Failure Modes:* AI errors  

- **Edit Fields3**  
  - *Type:* Set node  
  - *Role:* Assigns "Conclusion" field with AI output  
  - *Connections:* Outputs to Google Sheets2  
  - *Failure Modes:* None significant  

- **slug**  
  - *Type:* Langchain Agent  
  - *Role:* Create SEO-friendly URL slug based on blog post name  
  - *Connections:* Outputs to Edit Fields  
  - *Failure Modes:* AI errors, malformed slugs  

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Assigns "slug" field with AI output  
  - *Connections:* Outputs to Google Sheets2  
  - *Failure Modes:* None significant  

- **Google Sheets2**  
  - *Type:* Google Sheets Append  
  - *Role:* Final append of full blog post data (name, slug, intro, steps, conclusion) into "Blog" sheet  
  - *Config:* Same Google Sheet and sheet as Google Sheets1  
  - *Connections:* Outputs to Loop Over Items to continue processing  
  - *Failure Modes:* Auth errors, quota, data conflicts  

---

#### 1.5 Final Data Aggregation  
**Overview:**  
This block completes the loop by saving the fully generated blog post data back into the Google Sheet used for blog content, making it ready for publication or further editing.

**Nodes Involved:**  
- Google Sheets2  
- Loop Over Items (for continuing batch processing)  

**Node Details:**  

- **Google Sheets2**  
  - *As above in 1.4*  

- **Loop Over Items**  
  - *As above in 1.3*  
  - After appending, loops to process next question until all are done  

---

### 3. Summary Table

| Node Name                | Node Type                            | Functional Role                           | Input Node(s)                  | Output Node(s)                   | Sticky Note                                    |
|--------------------------|------------------------------------|-----------------------------------------|-------------------------------|---------------------------------|------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                     | Entry point to start workflow manually  |                               | Reddit                          | # Send data                                    |
| Reddit                   | Reddit node                        | Fetch latest 30 posts from subreddit    | When clicking ‘Test workflow’ | Code                           | # Send data                                    |
| Code                     | Code node (JavaScript)             | Filter posts to questions only          | Reddit                        | Google Sheets                  | # Data cleaner                                 |
| Google Sheets            | Google Sheets append               | Append filtered questions to sheet      | Code                         | Loop Over Items                | # Data cleaner                                 |
| Loop Over Items           | SplitInBatches                    | Process questions one by one             | Google Sheets                | AI Agent, (empty)              | # Enhancer                                     |
| AI Agent                 | Langchain Agent                    | Rephrase question using OpenAI           | Loop Over Items              | Edit Fields4                  | # Enhancer                                     |
| Edit Fields4             | Set node                         | Set blog post "name" field                | AI Agent                    | Google Sheets1                | # Enhancer                                     |
| Google Sheets1           | Google Sheets appendOrUpdate      | Store/update blog metadata                 | Edit Fields4                | Intro, Steps, Conclusion, slug | # Article factory                              |
| Intro                    | Langchain Agent                   | Generate blog introduction                 | Google Sheets1              | Edit Fields1                  | # Article factory                              |
| Edit Fields1             | Set node                         | Set "Intro" field                          | Intro                       | Google Sheets2                | # Article factory                              |
| Steps                    | Langchain Agent                   | Generate detailed steps section            | Google Sheets1              | Edit Fields2                  | # Article factory                              |
| Edit Fields2             | Set node                         | Set "Steps" field                          | Steps                       | Google Sheets2                | # Article factory                              |
| Conclusion               | Langchain Agent                   | Generate blog conclusion                   | Google Sheets1              | Edit Fields3                  | # Article factory                              |
| Edit Fields3             | Set node                         | Set "Conclusion" field                     | Conclusion                  | Google Sheets2                | # Article factory                              |
| slug                     | Langchain Agent                   | Generate SEO-friendly slug                 | Google Sheets1              | Edit Fields                  | # Article factory                              |
| Edit Fields              | Set node                         | Set "slug" field                           | slug                        | Google Sheets2                | # Article factory                              |
| Google Sheets2           | Google Sheets append              | Append final blog post data to sheet       | Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3 | Loop Over Items       | # Article factory                              |
| Simple Memory            | Langchain Memory Buffer Window   | Memory context for AI Agent (question rephrase) |                               | AI Agent                      |                                                |
| Simple Memory1           | Langchain Memory Buffer Window   | Memory for slug generation                 |                               | slug                         |                                                |
| Simple Memory2           | Langchain Memory Buffer Window   | Memory for intro generation                 |                               | Intro                        |                                                |
| Simple Memory3           | Langchain Memory Buffer Window   | Memory for steps generation                 |                               | Steps                        |                                                |
| Simple Memory4           | Langchain Memory Buffer Window   | Memory for conclusion generation            |                               | Conclusion                   |                                                |
| OpenAI Chat Model        | Langchain OpenAI Chat Model      | Provides GPT-4o-mini model for AI Agent    |                               | AI Agent                     |                                                |
| OpenAI Chat Model1       | Langchain OpenAI Chat Model      | GPT model for slug generation               |                               | slug                         |                                                |
| OpenAI Chat Model2       | Langchain OpenAI Chat Model      | GPT model for intro generation               |                               | Intro                        |                                                |
| OpenAI Chat Model3       | Langchain OpenAI Chat Model      | GPT model for steps generation               |                               | Steps                        |                                                |
| OpenAI Chat Model4       | Langchain OpenAI Chat Model      | GPT model for conclusion generation          |                               | Conclusion                   |                                                |
| Sticky Note              | Sticky Note                      | Visual note: "# Send data"                  |                               |                             | # Send data                                    |
| Sticky Note1             | Sticky Note                      | Visual note: "# Article factory"             |                               |                             | # Article factory                              |
| Sticky Note2             | Sticky Note                      | Visual note: "# Enhancer"                     |                               |                             | # Enhancer                                     |
| Sticky Note3             | Sticky Note                      | Visual note: "# Data cleaner"                  |                               |                             | # Data cleaner                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add Reddit Node**  
   - Name: "Reddit"  
   - Operation: Get All posts  
   - Subreddit: "n8n"  
   - Limit: 30  
   - Filters: Category "new"  
   - Credentials: Reddit OAuth2 account  
   - Connect output of Manual Trigger → Reddit.

3. **Add Code Node for Filtering Questions**  
   - Name: "Code"  
   - Type: JavaScript Code  
   - Paste filtering script that checks if titles are questions by identifying question words or '?' at end.  
   - Connect Reddit output → Code node.

4. **Add Google Sheets Append Node for "Queries"**  
   - Name: "Google Sheets"  
   - Operation: Append  
   - Document ID: Your Google Sheet ID with "Queries" sheet  
   - Sheet Name: Queries (gid=0)  
   - Columns: Map "Query" to `{{ $json.selftext }}`, "Title" to `{{ $json.title }}`  
   - Credentials: Google Sheets OAuth2 account  
   - Connect Code output → Google Sheets.

5. **Add SplitInBatches Node for Looping**  
   - Name: "Loop Over Items"  
   - Default batch size (1 recommended)  
   - Connect Google Sheets output → Loop Over Items.

6. **Add Langchain OpenAI Chat Model Node (GPT-4o-mini)**  
   - Name: "OpenAI Chat Model"  
   - Model: GPT-4o-mini  
   - Credentials: OpenAI API key

7. **Add Langchain Memory Buffer Window Node**  
   - Name: "Simple Memory"  
   - Session Key: `{{ $json.Title }}`  
   - Connect memory to AI Agent.

8. **Add Langchain Agent Node for Rephrasing Question**  
   - Name: "AI Agent"  
   - Prompt:  
     ```
     Here is a question: {{ $json.Title }}
     Rephrase it without changing the meaning. Keep it as a question.
     I just need the question as the output nothing else
     ```  
   - Connect Loop Over Items main output → AI Agent  
   - Connect OpenAI Chat Model → AI Agent language model input  
   - Connect Simple Memory → AI Agent memory input 

9. **Add Set Node to assign rephrased name**  
   - Name: "Edit Fields4"  
   - Set field: `name = {{ $json.output }}`  
   - Connect AI Agent → Edit Fields4

10. **Add Google Sheets AppendOrUpdate Node for "Blog" Metadata**  
    - Name: "Google Sheets1"  
    - Document ID: Same as earlier  
    - Sheet Name: Blog (gid=1732850028)  
    - Operation: Append or Update by "name" column  
    - Columns: name, slug, Intro, Steps, Conclusion (initially only name set)  
    - Credentials: Google Sheets OAuth2 account  
    - Connect Edit Fields4 → Google Sheets1

11. **For each blog section (slug, Intro, Steps, Conclusion), repeat:**  
    - Langchain OpenAI Chat Model node with GPT-4o-mini, named accordingly (OpenAI Chat Model1 to 4)  
    - Langchain Memory Buffer Window node (Simple Memory1 to 4) with sessionKey = name or relevant field  
    - Langchain Agent node with prompts:  
      - slug: "Create a website slug for {{ $json.name }}..."  
      - Intro: "Write a short intro for a blog post titled {{ $json.name }}..."  
      - Steps: "Write a 'step by step guide' for {{ $json.name }}..."  
      - Conclusion: "Write a short conclusion for {{ $json.name }}..."  
    - Set node to assign their outputs to respective fields: slug, Intro, Steps, Conclusion  
    - Connect Google Sheets1 output → each Langchain Agent → corresponding Set node

12. **Add Google Sheets Append Node for final blog posts**  
    - Name: "Google Sheets2"  
    - Document ID and Sheet Name same as Google Sheets1  
    - Operation: Append  
    - Columns: name, slug, Intro, Steps, Conclusion (populated from Set nodes)  
    - Credentials: Google Sheets OAuth2 account  
    - Connect all Edit Fields nodes (slug, Intro, Steps, Conclusion) → Google Sheets2  

13. **Connect Google Sheets2 output back to Loop Over Items (empty output) to continue processing batches until complete.**

14. **Add Sticky Notes for Visual Organization** (optional)  
    - "# Send data" near initial nodes  
    - "# Data cleaner" near Code node  
    - "# Enhancer" near AI Agent nodes  
    - "# Article factory" near blog content generation nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow transforms Reddit questions into SEO blog posts using AI and Google Sheets.         | Core purpose overview                                                                                     |
| Uses GPT-4o-mini OpenAI model for efficient language generation within n8n Langchain nodes.  | Model choice for balancing performance and cost                                                        |
| Source Google Sheet URL (VOC to blog): https://docs.google.com/spreadsheets/d/1fIk66lycsodCwVWY_3Jz8ReQBeo0_nOHaFuv427pzKQ | Main data repository for queries and blog content                                                       |
| Refer to n8n documentation for setting up OAuth2 credentials for Reddit and Google Sheets.   | Credential setup: https://docs.n8n.io/credentials/oauth2/                                                |
| Langchain integration in n8n enables prompt chaining with memory buffers for context handling.| Useful for iterative AI content generation workflows                                                    |
| Sticky notes are used in the workflow for clarity: "Send data", "Data cleaner", "Enhancer", "Article factory". | Visual grouping of logical blocks                                                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.