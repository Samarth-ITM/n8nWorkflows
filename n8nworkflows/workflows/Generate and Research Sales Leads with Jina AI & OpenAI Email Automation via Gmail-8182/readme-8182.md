Generate and Research Sales Leads with Jina AI & OpenAI Email Automation via Gmail

https://n8nworkflows.xyz/workflows/generate-and-research-sales-leads-with-jina-ai---openai-email-automation-via-gmail-8182


# Generate and Research Sales Leads with Jina AI & OpenAI Email Automation via Gmail

### 1. Workflow Overview

This workflow automates the process of generating and researching sales leads using AI-powered data extraction and email drafting, integrating Jina AI, OpenAI language models, Airtable for data storage, and Gmail for email automation. It is designed for sales or marketing teams needing to enrich lead information, analyze it with AI, and create personalized emails to engage prospects efficiently.

The workflow is logically grouped into the following blocks:

- **1.1 Initialization & Data Retrieval:** Loading business parameters and API keys; fetching input lead records from Airtable.
- **1.2 Lead Data Processing & Research:** Looping over each lead, enriching and researching lead details via Jina AI and HTTP requests.
- **1.3 AI Analysis & Lead Qualification:** Using Langchain agents and OpenAI models to analyze leads, parse structured outputs, and evaluate data quality.
- **1.4 Email Content Generation and Refinement:** Creating initial email drafts with AI, rewriting content for improvements, formatting HTML emails.
- **1.5 Email Draft Creation & Record Updates:** Drafting the email in Gmail, updating Airtable records with results, and implementing wait times for API rate limits and sequencing.
- **1.6 Control Flow & Error Handling:** Conditional checks and merges to coordinate data flow and handle branching logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Data Retrieval

**Overview:**  
This block initializes workflow execution by setting essential parameters like business info and API keys. Then, it retrieves the raw lead data from Airtable to process.

**Nodes Involved:**  
- Execute workflow (Manual Trigger)  
- Schedule Trigger  
- Business_Info (Set)  
- Jina_API_Key (Set)  
- Get input records (Airtable)

**Node Details:**

- **Execute workflow (Manual Trigger)**  
  - Type: Trigger node  
  - Role: Manual start for testing or on-demand runs  
  - Output: Starts Business_Info node  
  - Edge cases: Manual trigger requires user interaction; no automatic scheduling

- **Schedule Trigger**  
  - Type: Trigger node  
  - Role: Scheduled execution (e.g., periodic runs)  
  - Output: Starts Business_Info node  
  - Edge cases: Schedule misconfiguration may cause missed runs

- **Business_Info (Set)**  
  - Type: Set node  
  - Role: Defines business-related parameters (e.g., company name, industry) used downstream  
  - Output: Passes data to Jina_API_Key node  
  - Edge cases: Missing or incorrect business info can impact lead enrichment accuracy

- **Jina_API_Key (Set)**  
  - Type: Set node  
  - Role: Holds credentials/API key for Jina AI calls  
  - Output: Passes data to Get input records  
  - Edge cases: Missing or invalid API key causes authentication failures in Jina requests

- **Get input records (Airtable)**  
  - Type: Airtable node  
  - Role: Retrieves lead records from Airtable base/table configured  
  - Output: Provides raw leads to Loop Over Items node  
  - Edge cases: Airtable credential or permission errors; empty or malformed datasets

---

#### 1.2 Lead Data Processing & Research

**Overview:**  
Iterates over each lead record in batches; performs HTTP requests to research individual lead persons; merges research results for analysis.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Lead Person Research (HTTP Request)  
- Merge-Search (Merge)

**Node Details:**

- **Loop Over Items (SplitInBatches)**  
  - Type: SplitInBatches node  
  - Role: Processes lead records in batches (default batch size if configured) to manage load  
  - Inputs: Lead records from Airtable  
  - Outputs: Sends batches to Lead Person Research and Merge-loop nodes (parallel branches)  
  - Edge cases: Batch size misconfiguration can cause memory or rate limit issues

- **Lead Person Research (HTTP Request)**  
  - Type: HTTP Request node  
  - Role: Queries external API (likely Jina or other research API) to enrich lead person data  
  - Configuration: Uses method, URL, headers (including API key from Jina_API_Key node), and dynamic parameters from input lead data  
  - Inputs: Lead data from batches  
  - Outputs: Enriched lead data to Merge-Search  
  - Edge cases: HTTP errors, timeout, invalid responses, API quota exceeded

- **Merge-Search (Merge)**  
  - Type: Merge node  
  - Role: Combines outputs from Lead Person Research and Lead Analyzer nodes  
  - Inputs: Waits for data from both sources  
  - Outputs: Passes merged data to Lead Analyzer for AI processing  
  - Edge cases: Mismatched data sizes can cause merge failures

---

#### 1.3 AI Analysis & Lead Qualification

**Overview:**  
Uses Langchain agents powered by OpenAI GPT-5-mini models to analyze merged lead data, parse structured outputs, evaluate lead quality, and prepare for email generation.

**Nodes Involved:**  
- Lead Analyzer (@n8n/n8n-nodes-langchain.agent)  
- Structured Output Parser (x4) (@n8n/n8n-nodes-langchain.outputParserStructured)  
- OpenAI Gpt-5-mini (LM Chat OpenAI)  
- OpenAI - Gpt-5-mini-hi (LM Chat OpenAI)  
- OpenAI-Gpt-5-mini-low (LM Chat OpenAI)  
- Window Buffer Memory (@n8n/n8n-nodes-langchain.memoryBufferWindow)  
- Evaluator (@n8n/n8n-nodes-langchain.agent)  
- If (Conditional)  
- Feedback (Set)

**Node Details:**

- **Lead Analyzer**  
  - Type: Langchain Agent  
  - Role: Uses AI model to analyze lead data and generate insights or scores  
  - Configuration: Uses OpenAI GPT-5-mini as language model with Structured Output Parser  
  - Inputs: Merged lead research data  
  - Outputs: Structured insights to Wait-5-sec node  
  - Edge cases: AI API quota, response latency, malformed outputs

- **Structured Output Parser (4 Instances)**  
  - Type: Output Parser  
  - Role: Parses AI-generated JSON or structured text into usable data fields  
  - Usage: One parser per AI interaction stage (Lead Analyzer, Evaluator, Email Content Creator, Email Content Re-writer)  
  - Edge cases: Parsing failures if AI output format changes or is invalid

- **OpenAI GPT-5-mini / Variants**  
  - Type: Language Model Chat nodes  
  - Role: AI models used in different contexts — analyzing lead, creating email content, rewriting email, formatting HTML  
  - Configuration: API key, model version, temperature, max tokens configured as per node  
  - Edge cases: API quota, latency, model version compatibility

- **Window Buffer Memory**  
  - Type: Memory Buffer for Langchain  
  - Role: Maintains context window for Evaluator agent to improve contextual understanding  
  - Inputs: Feeds from Email Content Creator  
  - Outputs: Feeds Evaluator agent  
  - Edge cases: Memory overflow or context loss if buffer too small or too large

- **Evaluator**  
  - Type: Langchain Agent  
  - Role: Evaluates generated email content, decides if re-writing is needed based on quality or criteria  
  - Inputs: Output from Email Content Creator and Window Buffer Memory  
  - Outputs: Decision to If node for branching  
  - Edge cases: False positives/negatives in quality assessment

- **If (Conditional)**  
  - Type: Conditional branching node  
  - Role: Routes flow depending on evaluation results — either to Format HTML or Feedback for rewriting  
  - Edge cases: Expression evaluation errors; misrouting

- **Feedback (Set)**  
  - Type: Set node  
  - Role: Prepares feedback data or flags for email content rewriting  
  - Outputs: To Email Content Re-writer node  
  - Edge cases: Missing feedback data causing rewrite failures

---

#### 1.4 Email Content Generation and Refinement

**Overview:**  
Creates sales email content using AI, rewrites as necessary, formats into HTML, and prepares drafts in Gmail.

**Nodes Involved:**  
- Email Content Creator (@n8n/n8n-nodes-langchain.agent)  
- Email Content Re-writer (@n8n/n8n-nodes-langchain.agent)  
- Format HTML (@n8n/n8n-nodes-langchain.agent)  
- Structured Output Parser3 and 4  
- OpenAI language model nodes for content creation and rewriting  
- Create a draft (Gmail)

**Node Details:**

- **Email Content Creator**  
  - Type: Langchain Agent  
  - Role: Generates initial draft of the sales email based on analyzed lead data  
  - Inputs: Wait-5-sec output from Lead Analyzer  
  - Outputs: To Evaluator for quality check  
  - Edge cases: Output irrelevance or incoherence

- **Email Content Re-writer**  
  - Type: Langchain Agent  
  - Role: Modifies and improves email content per feedback  
  - Inputs: Feedback node output  
  - Outputs: Back to Evaluator for re-assessment  
  - Edge cases: Endless rewrite loops if criteria unclear

- **Format HTML**  
  - Type: Langchain Agent  
  - Role: Converts plain text email into formatted HTML content suitable for Gmail drafts  
  - Inputs: If node’s positive branch (approved content)  
  - Outputs: To Create a draft (Gmail)  
  - Edge cases: Formatting errors or output incompatibilities

- **Create a draft (Gmail)**  
  - Type: Gmail node  
  - Role: Creates email draft in Gmail account with formatted content  
  - Configuration: Uses OAuth2 credentials for Gmail  
  - Edge cases: Auth failures, Gmail API quota, draft creation errors

---

#### 1.5 Email Draft Creation & Record Updates

**Overview:**  
Finalizes the lead processing by updating Airtable with new information and manages wait times to avoid API rate limits.

**Nodes Involved:**  
- Merge-loop (Merge)  
- Update record (Airtable)  
- Wait-5-sec (Wait)  
- Wait-8-sec (Wait)

**Node Details:**

- **Merge-loop (Merge)**  
  - Type: Merge node  
  - Role: Combines batch processing outputs before Airtable update  
  - Inputs: From Loop Over Items and Create a draft outputs  
  - Edge cases: Data mismatches causing merge failures

- **Update record (Airtable)**  
  - Type: Airtable node  
  - Role: Updates Airtable records with enriched lead data and email draft status  
  - Edge cases: Airtable API errors, permission issues

- **Wait-5-sec and Wait-8-sec**  
  - Type: Wait nodes  
  - Role: Introduce delays to respect API rate limits and processing order  
  - Edge cases: Excessive delays may impact throughput; missing waits may cause rate limiting

---

#### 1.6 Control Flow & Error Handling

**Overview:**  
Implements logical branching and merges to control workflow execution and handle conditional processing paths.

**Nodes Involved:**  
- If (Conditional)  
- Merge-Search  
- Merge-loop

**Node Details:**

- **If (Conditional)**  
  - Routes based on Evaluator’s output whether to format final email or send back for rewriting

- **Merge-Search and Merge-loop**  
  - Ensure synchronization of parallel branches and data integrity

- Edge cases: Incorrect expressions, data mismatches, or missing branches can cause workflow failures or incomplete processing.

---

### 3. Summary Table

| Node Name            | Node Type                                   | Functional Role                            | Input Node(s)                | Output Node(s)               | Sticky Note                                            |
|----------------------|---------------------------------------------|--------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------|
| Execute workflow      | Manual Trigger                             | Manual start trigger                       | —                           | Business_Info               |                                                        |
| Schedule Trigger      | Schedule Trigger                           | Scheduled start trigger                    | —                           | Business_Info               |                                                        |
| Business_Info        | Set                                        | Sets business parameters                   | Execute workflow, Schedule Trigger | Jina_API_Key                |                                                        |
| Jina_API_Key          | Set                                        | Stores Jina AI API key                     | Business_Info               | Get input records           |                                                        |
| Get input records     | Airtable                                   | Retrieves lead records                     | Jina_API_Key                | Loop Over Items             |                                                        |
| Loop Over Items       | SplitInBatches                             | Batch processing of leads                  | Get input records           | Lead Person Research, Merge-Search, Merge-loop (parallel) |                                                        |
| Lead Person Research  | HTTP Request                              | Enriches lead data via external API        | Loop Over Items (batch)      | Merge-Search               |                                                        |
| Merge-Search          | Merge                                      | Combines research and analysis data        | Lead Person Research, Lead Analyzer | Lead Analyzer               |                                                        |
| Lead Analyzer         | Langchain Agent                            | AI analysis of leads                        | Merge-Search                | Wait-5-sec                 |                                                        |
| Wait-5-sec            | Wait                                       | Delay for API rate limit                    | Lead Analyzer               | Email Content Creator      |                                                        |
| Email Content Creator | Langchain Agent                            | Generates initial email content             | Wait-5-sec                  | Evaluator                  |                                                        |
| Window Buffer Memory  | Langchain Memory Buffer                    | Maintains AI context for evaluation        | Email Content Creator       | Evaluator                  |                                                        |
| Evaluator             | Langchain Agent                            | Evaluates email content quality             | Email Content Creator, Window Buffer Memory | If                        |                                                        |
| If                    | Conditional                               | Branches based on evaluation result         | Evaluator                   | Format HTML, Feedback       |                                                        |
| Feedback              | Set                                        | Prepares feedback for rewriting             | If                         | Email Content Re-writer    |                                                        |
| Email Content Re-writer | Langchain Agent                          | Rewrites email content                      | Feedback                    | Evaluator                  |                                                        |
| Format HTML           | Langchain Agent                            | Formats email content to HTML                | If                         | Create a draft             |                                                        |
| Create a draft        | Gmail                                      | Creates Gmail draft email                    | Format HTML                 | Merge-loop                 |                                                        |
| Merge-loop            | Merge                                      | Merges final outputs                         | Loop Over Items, Create a draft | Update record             |                                                        |
| Update record         | Airtable                                   | Updates Airtable with enriched data         | Merge-loop                  | Wait-8-sec                 |                                                        |
| Wait-8-sec            | Wait                                       | Delay for sequencing                         | Update record               | Loop Over Items            |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `Execute workflow` for manual starts.  
   - Add a **Schedule Trigger** node named `Schedule Trigger` for periodic runs.

2. **Set Business Parameters**  
   - Add a **Set** node named `Business_Info`. Configure to store business-specific parameters (e.g., company name, industry). Connect both trigger nodes to this node.

3. **Set Jina AI API Key**  
   - Add a **Set** node named `Jina_API_Key` to hold the Jina API key securely. Connect `Business_Info` to this node.

4. **Retrieve Lead Records**  
   - Add an **Airtable** node named `Get input records`. Configure with Airtable credentials and specify the base and table containing lead records. Connect `Jina_API_Key` to this node.

5. **Batch Processing Loop**  
   - Add a **SplitInBatches** node named `Loop Over Items`. Connect `Get input records` to it. Use default batch size or adjust as needed.

6. **Lead Person Research HTTP Request**  
   - Add an **HTTP Request** node named `Lead Person Research`. Configure it to query the external research API (e.g., Jina AI). Use parameters from the batch item (e.g., lead email or name). Include authentication headers using the API key from `Jina_API_Key`. Connect `Loop Over Items` to this node.

7. **Merge Research and Analysis Data**  
   - Add a **Merge** node named `Merge-Search`. Connect `Lead Person Research` and the upcoming `Lead Analyzer` node outputs to this node.

8. **Lead Analyzer AI Node**  
   - Add a **Langchain Agent** node named `Lead Analyzer`. Configure it to use OpenAI GPT-5-mini model. Attach a **Structured Output Parser** node to parse its output. Connect `Merge-Search` to `Lead Analyzer`.

9. **Wait 5 Seconds**  
   - Add a **Wait** node named `Wait-5-sec`. Connect `Lead Analyzer` to this node.

10. **Email Content Creator AI Node**  
    - Add a **Langchain Agent** node named `Email Content Creator`. Configure it with OpenAI GPT-5-mini-hi model for generating email content. Connect `Wait-5-sec` to this node. Attach a **Structured Output Parser3** to parse the output.

11. **Window Buffer Memory**  
    - Add a **Memory Buffer Window** node named `Window Buffer Memory` to maintain context. Connect `Email Content Creator` to this node.

12. **Evaluator AI Node**  
    - Add a **Langchain Agent** named `Evaluator`. Use OpenAI GPT-5-mini-low model. Connect both `Email Content Creator` and `Window Buffer Memory` outputs as inputs to `Evaluator`. Attach a **Structured Output Parser2** to parse evaluation results.

13. **Conditional Branching**  
    - Add an **If** node named `If`. Configure it to branch based on `Evaluator` output (e.g., quality threshold). Connect `Evaluator` to `If`.

14. **Feedback Preparation**  
    - Add a **Set** node named `Feedback` for the feedback data used in rewriting. Connect the false branch of `If` to this node.

15. **Email Content Re-writer**  
    - Add a **Langchain Agent** node named `Email Content Re-writer`. Use OpenAI GPT-5-mini-low model. Connect `Feedback` to this node. Attach **Structured Output Parser4**. Connect `Email Content Re-writer` output back to `Evaluator` for re-evaluation (loop).

16. **Format Email to HTML**  
    - Add a **Langchain Agent** node named `Format HTML`. Use OpenAI GPT-5-mini-low1 model. Connect the true branch of `If` to this node.

17. **Create Gmail Draft**  
    - Add a **Gmail** node named `Create a draft`. Configure with OAuth2 credentials for Gmail. Map the formatted HTML email content as the draft body. Connect `Format HTML` to this node.

18. **Merge Loop Outputs**  
    - Add a **Merge** node named `Merge-loop`. Connect both `Loop Over Items` (second output) and `Create a draft` node to it.

19. **Update Airtable Records**  
    - Add an **Airtable** node named `Update record`. Configure it to update processed lead records with enriched data and email status. Connect `Merge-loop` to this node.

20. **Wait 8 Seconds**  
    - Add a **Wait** node named `Wait-8-sec`. Connect `Update record` to this node.

21. **Loop Continuation**  
    - Connect `Wait-8-sec` back to `Loop Over Items` to process the next batch.

22. **Configure Credentials**  
    - Ensure Airtable credentials are set with correct API keys and permissions.  
    - Configure OpenAI credentials with valid API keys and model access.  
    - Configure Gmail OAuth2 credentials with permission to create drafts.

23. **Verify & Test**  
    - Test manual trigger first to verify data flow and outputs.  
    - Confirm scheduled trigger runs successfully.  
    - Monitor API quotas and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses advanced Langchain nodes integrated into n8n for AI-powered processing.   | See https://docs.n8n.io/integrations/builtin/ai/ for Langchain integration details                  |
| Gmail node requires OAuth2 credentials with scope to create drafts and send emails.          | https://developers.google.com/gmail/api/auth/scopes                                                |
| Airtable node requires API key and proper base/table permissions for read and write access.  | https://airtable.com/api                                                                           |
| Jina AI API key must be valid and have sufficient quota for HTTP requests used in research.  | https://docs.jina.ai/api/                                                                           |
| Rate limiting is handled via Wait nodes (5 sec and 8 sec) to prevent API throttling.          | Adjust wait times as per API limits                                                               |
| The conditional logic ensures email content quality by looping through rewriting as needed. | Avoid infinite loops by setting max retry limits or conditions                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.