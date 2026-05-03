Generate Sales Leads & Personalized Outreach Emails using Jina AI and OpenAI Agents

https://n8nworkflows.xyz/workflows/generate-sales-leads---personalized-outreach-emails-using-jina-ai-and-openai-agents-6649


# Generate Sales Leads & Personalized Outreach Emails using Jina AI and OpenAI Agents

### 1. Workflow Overview

This workflow automates the generation of sales leads and the creation of personalized outreach emails using AI technologies — specifically Jina AI for research and OpenAI-based agents for analysis, content creation, and evaluation. Its core purpose is to streamline sales prospecting by identifying the best contact persons within target companies and automatically drafting and assessing marketing emails tailored to them.

The workflow is logically divided into the following blocks:

- **1.1 Configuration and Input Loading**  
  Initialization of business context and retrieving unprocessed company records from Airtable.

- **1.2 Deep Research with Jina AI**  
  Conducting in-depth research on each company to discover decision-makers and relevant lead information.

- **1.3 Lead Analysis with AI Agent**  
  Processing deep research results to select the most suitable lead contact for outreach.

- **1.4 Email Content Creation with AI Agent**  
  Generating a personalized, persuasive marketing email and subject line based on lead and business data.

- **1.5 Email Evaluation with AI Agent**  
  Evaluating the generated email for quality, tone, and effectiveness to ensure high impact.

- **1.6 Updating Airtable Records**  
  Writing back the enriched lead and email data into Airtable and marking records as processed.

These blocks work sequentially with batching and merging nodes to handle multiple records efficiently.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration and Input Loading

**Overview:**  
Sets up static business data as context for AI agents and fetches unprocessed company leads from Airtable.

**Nodes involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- Business_Info (Set)  
- Jina_API_Key (Set)  
- Get input records (Airtable)

**Node Details:**

- **When clicking "Test workflow"**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing or execution.  
  - Config: No parameters.  
  - Inputs: None  
  - Outputs: Triggers Business_Info node.  
  - Edge cases: None.

- **Business_Info**  
  - Type: Set  
  - Role: Stores company's static info and marketing context (business name, key benefits, target audience, landing page URL).  
  - Config: Hardcoded values for fields like BUSINESS_NAME = "ACME SA", BUSINESS_INFORMATION, BUSINESS_KEY_BENEFITS (detailed list), LANDING_PAGE_URL, LEAD_TARGET_AUDIENCE.  
  - Inputs: Trigger from manual node.  
  - Outputs: Triggers Jina_API_Key node.  
  - Edge cases: None; ensure info is accurate for best AI output.

- **Jina_API_Key**  
  - Type: Set  
  - Role: Holds Jina AI API key for authentication in research requests.  
  - Config: String field `jina_api` (empty by default, user must fill).  
  - Inputs: From Business_Info.  
  - Outputs: Triggers Get input records.  
  - Edge cases: Missing or invalid API key will cause auth errors downstream.

- **Get input records**  
  - Type: Airtable  
  - Role: Queries Airtable base/table for records where `processed` field = "no" to find companies not yet handled.  
  - Config: Base and table selected; filter formula: `{processed} = "no"`.  
  - Inputs: From Jina_API_Key.  
  - Outputs: Triggers Loop Over Items.  
  - Edge cases: No records found results in no further processing. Airtable API limits or auth errors possible.

---

#### 1.2 Deep Research with Jina AI

**Overview:**  
For each company record, performs a deep research query to Jina AI's chat completion endpoint to find decision-makers based on company name and website.

**Nodes involved:**  
- Loop Over Items (SplitInBatches)  
- Lead Person Research (HTTP Request)  
- Merge-Search (Merge)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes companies one-by-one or in batches (default batch size 1) for rate limiting and orderly processing.  
  - Config: `reset` option false to keep state.  
  - Inputs: From Get input records.  
  - Outputs: Branch 1 (empty) and Branch 2 triggers Lead Person Research and Merge-Search.  
  - Edge cases: Batch processing failures or timeouts.

- **Lead Person Research**  
  - Type: HTTP Request  
  - Role: Sends POST request to `https://deepsearch.jina.ai/v1/chat/completions` with a JSON payload instructing the model to find sales leads (name, role, email, summary) without hallucination.  
  - Config:  
    - Uses `jina_api` token from Jina_API_Key node for Authorization header.  
    - Timeout set to 240 seconds for potentially slow responses.  
    - JSON body includes dynamic company fields `Company_name` and `Company_website`.  
  - Inputs: From Loop Over Items.  
  - Outputs: Triggers Merge-Search.  
  - Version: Uses HTTP request version 4.2 supporting advanced features.  
  - Edge cases: Timeout, auth failure, empty or null response; missing company data may lead to poor results.

- **Merge-Search**  
  - Type: Merge  
  - Role: Combines outputs from Lead Person Research and Loop Over Items to enable parallel processing downstream.  
  - Config: Mode `combine` with `combineAll` option to merge all incoming data.  
  - Inputs: From Lead Person Research and Loop Over Items.  
  - Outputs: Triggers Lead Analyzer.  
  - Edge cases: Mismatched inputs causing merge failures.

---

#### 1.3 Lead Analysis with AI Agent

**Overview:**  
Analyzes the deep research output and company info to identify the best lead contact person for outreach using a Langchain agent with OpenAI.

**Nodes involved:**  
- Lead Analyzer (Langchain Agent)  
- Structured Output Parser (Langchain Output Parser)  
- OpenAI o3-mini (OpenAI Chat Model)  
- Wait-5-sec (Wait)  

**Node Details:**

- **Lead Analyzer**  
  - Type: Langchain Agent  
  - Role: Processes research results and business info to select lead_name, lead_email, and summarize reasoning.  
  - Config:  
    - Input prompt combines company info and deep research content.  
    - Output expected in JSON format with specified fields.  
    - Max iterations limited to 4 for efficiency.  
    - System message defines role as Lead Generation Specialist with hallucination reduction emphasis.  
  - Inputs: From Merge-Search node.  
  - Outputs: To Structured Output Parser and then Wait-5-sec.  
  - Edge cases: AI hallucination, incomplete data; parser failures if output not well-formed.

- **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Parses Lead Analyzer JSON output to structured fields.  
  - Config: JSON schema example with company_name, lead_email, lead_name, summary_of_your_though fields.  
  - Inputs: From Lead Analyzer.  
  - Outputs: To Wait-5-sec.  
  - Edge cases: Parsing errors on malformed AI output.

- **OpenAI o3-mini**  
  - Type: OpenAI Chat Model  
  - Role: Provides language model backend for Lead Analyzer agent.  
  - Config: Model "o3-mini", default options.  
  - Inputs: Connected internally to Lead Analyzer.  
  - Outputs: To Lead Analyzer.  
  - Edge cases: API quota limits, network errors.

- **Wait-5-sec**  
  - Type: Wait  
  - Role: Throttles processing to avoid API rate limits before next step.  
  - Config: Wait time 5 seconds.  
  - Inputs: From Lead Analyzer.  
  - Outputs: To Email Content Creator.  
  - Edge cases: None.

---

#### 1.4 Email Content Creation with AI Agent

**Overview:**  
Creates a personalized marketing email and subject line tailored to the identified lead using business info and lead analysis results.

**Nodes involved:**  
- Email Content Creator (Langchain Agent)  
- Structured Output Parser3 (Langchain Output Parser)  
- OpenAI - 4o-mini (OpenAI Chat Model)

**Node Details:**

- **Email Content Creator**  
  - Type: Langchain Agent  
  - Role: Drafts a professional, friendly, persuasive, and concise marketing email and subject line targeting the lead.  
  - Config:  
    - Prompt includes business info, lead info, key benefits, landing page URL, and expected output fields.  
    - System message instructs to produce complete emails without placeholders and reduce hallucinations.  
  - Inputs: From Wait-5-sec.  
  - Outputs: To Structured Output Parser3 and then Evaluator.  
  - Edge cases: AI hallucination, incomplete or inappropriate email content.

- **Structured Output Parser3**  
  - Type: Langchain Output Parser  
  - Role: Parses JSON email output (contact_name, contact_email, email_subject, email_wrote, your_summary).  
  - Inputs: From Email Content Creator.  
  - Outputs: To Evaluator node.  
  - Edge cases: Parsing failures if output malformed.

- **OpenAI - 4o-mini**  
  - Type: OpenAI Chat Model  
  - Role: Language model backend for Email Content Creator.  
  - Config: Temperature 0.8 for creative output.  
  - Inputs: Internal to Email Content Creator.  
  - Edge cases: API limits or latency.

---

#### 1.5 Email Evaluation with AI Agent

**Overview:**  
Assesses the quality and effectiveness of the generated email to provide feedback, scoring, and suggestions for improvement.

**Nodes involved:**  
- Evaluator (Langchain Agent)  
- Structured Output Parser1 (Langchain Output Parser)  
- OpenAI - 4o-mini - low (OpenAI Chat Model)  
- Merge-loop (Merge)  
- Wait-8-sec (Wait)

**Node Details:**

- **Evaluator**  
  - Type: Langchain Agent  
  - Role: Expert Email Marketing Evaluator reviews the marketing email based on criteria such as subject effectiveness, content quality, readability, call-to-action strength, and technical correctness.  
  - Config:  
    - Input includes original brief, target audience, key benefits, call-to-action, and the generated email.  
    - Output JSON includes contact info, email subject, email content, evaluation summary, overall score, and areas for improvement.  
    - Max iterations 3 for iterative refinement.  
    - System message defines detailed evaluation framework and instructions.  
  - Inputs: From Structured Output Parser3.  
  - Outputs: To Structured Output Parser1 and Merge-loop.  
  - Edge cases: Parsing errors, AI output issues.

- **Structured Output Parser1**  
  - Type: Langchain Output Parser  
  - Role: Parses evaluation output JSON into structured fields.  
  - Inputs: From Evaluator.  
  - Outputs: To Merge-loop.  
  - Edge cases: Parsing failures.

- **OpenAI - 4o-mini - low**  
  - Type: OpenAI Chat Model  
  - Role: Language model backend for Evaluator with temperature 0.3 for more focused output.  
  - Inputs: Internal to Evaluator.  
  - Edge cases: API quota, latency.

- **Merge-loop**  
  - Type: Merge  
  - Role: Combines data streams to prepare for final update.  
  - Config: Mode `combine` with `combineAll`.  
  - Inputs: From Loop Over Items and Evaluator.  
  - Outputs: To Update record.  
  - Edge cases: Merge mismatches.

- **Wait-8-sec**  
  - Type: Wait  
  - Role: Delays workflow to avoid Airtable or API rate limits before the next batch cycle.  
  - Inputs: From Update record.  
  - Outputs: To Loop Over Items to continue processing next batch.  
  - Edge cases: None.

---

#### 1.6 Updating Airtable Records

**Overview:**  
Updates the original Airtable record with generated lead details, email content, evaluation results, and marks the record as processed.

**Nodes involved:**  
- Update record (Airtable)

**Node Details:**

- **Update record**  
  - Type: Airtable  
  - Role: Writes back to Airtable table, updating fields: processed = "yes", lead_name, lead_email, email_subject, email_text, email_summary, create_date (current timestamp), task_result ("Email wrote").  
  - Config: Uses record ID from input data to update correct record.  
  - Inputs: From Merge-loop.  
  - Outputs: To Wait-8-sec for batch continuation.  
  - Edge cases: Airtable API limits, auth errors, write conflicts.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                             | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                           |
|-------------------------|---------------------------------|---------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger                  | Starts workflow manually                     | None                        | Business_Info               |                                                                                                     |
| Business_Info           | Set                             | Stores business static data                   | When clicking "Test workflow" | Jina_API_Key                | See Sticky Note: Configuration includes business details, benefits, target audience, landing page.  |
| Jina_API_Key            | Set                             | Stores Jina AI API key                        | Business_Info               | Get input records           | See Sticky Note: Save your Jina API key here.                                                       |
| Get input records       | Airtable                        | Fetches unprocessed company records          | Jina_API_Key                | Loop Over Items             | See Sticky Note: Loads lead info from Airtable including company name, website, email, processed.   |
| Loop Over Items         | SplitInBatches                  | Batches companies for processing              | Get input records           | Lead Person Research, Merge-Search, Merge-loop |                                                                                                     |
| Lead Person Research    | HTTP Request                   | Performs deep research on company leads       | Loop Over Items             | Merge-Search                | See Sticky Note: Deep research step using Jina AI HTTP request.                                     |
| Merge-Search            | Merge                          | Combines research output with input batch     | Lead Person Research, Loop Over Items | Lead Analyzer              |                                                                                                     |
| Lead Analyzer           | Langchain Agent                | Analyzes research to select best lead         | Merge-Search                | Structured Output Parser, Wait-5-sec | See Sticky Note: Lead analysis using AI agent to extract best lead info.                            |
| Structured Output Parser| Langchain Output Parser        | Parses lead analyzer JSON output               | Lead Analyzer               | Wait-5-sec                  |                                                                                                     |
| OpenAI o3-mini          | OpenAI Chat Model              | Language model for Lead Analyzer               | Internal to Lead Analyzer   | Lead Analyzer               |                                                                                                     |
| Wait-5-sec              | Wait                           | Waits 5 seconds between steps                  | Structured Output Parser    | Email Content Creator       |                                                                                                     |
| Email Content Creator   | Langchain Agent                | Generates personalized marketing email         | Wait-5-sec                 | Structured Output Parser3, Evaluator | See Sticky Note: Email content creation agent using business and lead info.                         |
| Structured Output Parser3| Langchain Output Parser       | Parses email creation output                    | Email Content Creator       | Evaluator                   |                                                                                                     |
| OpenAI - 4o-mini        | OpenAI Chat Model              | Language model for Email Content Creator       | Internal to Email Content Creator | Email Content Creator       |                                                                                                     |
| Evaluator               | Langchain Agent                | Evaluates generated email for quality          | Structured Output Parser3   | Structured Output Parser1, Merge-loop | See Sticky Note: Email evaluation agent providing detailed feedback and scoring.                    |
| Structured Output Parser1| Langchain Output Parser       | Parses email evaluation output                   | Evaluator                  | Merge-loop                  |                                                                                                     |
| OpenAI - 4o-mini - low  | OpenAI Chat Model              | Language model for Evaluator                     | Internal to Evaluator       | Evaluator                   |                                                                                                     |
| Merge-loop              | Merge                          | Combines evaluation and batch data              | Loop Over Items, Evaluator  | Update record               |                                                                                                     |
| Update record           | Airtable                       | Updates Airtable with lead and email data       | Merge-loop                  | Wait-8-sec                  | See Sticky Note: Writes back lead name, email, email content, evaluation summary, marks processed. |
| Wait-8-sec              | Wait                           | Waits 8 seconds between batch cycles             | Update record               | Loop Over Items             |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Add "Manual Trigger" node named "When clicking \"Test workflow\"".

2. **Set Business Info**  
   - Add "Set" node named "Business_Info".  
   - Add string fields:  
     - BUSINESS_NAME: e.g., "ACME SA"  
     - BUSINESS_INFORMATION: Company overview text  
     - BUSINESS_KEY_BENEFITS: List key benefits separated by line breaks  
     - LANDING_PAGE_URL: e.g., "http://acme.com"  
     - LEAD_TARGET_AUDIENCE: e.g., "Startups"  
   - Connect "When clicking \"Test workflow\"" → "Business_Info".

3. **Set Jina API Key**  
   - Add "Set" node named "Jina_API_Key".  
   - Add string field: `jina_api` for Jina AI API key (user must fill).  
   - Connect "Business_Info" → "Jina_API_Key".

4. **Get Unprocessed Records from Airtable**  
   - Add "Airtable" node named "Get input records".  
   - Configure credentials for your Airtable account.  
   - Select the base and table containing company leads.  
   - Set operation to "Search" with filter formula `{processed} = "no"`.  
   - Connect "Jina_API_Key" → "Get input records".

5. **Batch Processing Setup**  
   - Add "SplitInBatches" node named "Loop Over Items".  
   - Configure with default batch size (1).  
   - Connect "Get input records" → "Loop Over Items".

6. **Deep Research HTTP Request**  
   - Add "HTTP Request" node named "Lead Person Research".  
   - Set method POST, URL `https://deepsearch.jina.ai/v1/chat/completions`.  
   - Headers: Authorization `Bearer {{ $('Jina_API_Key').item.json.jina_api }}`.  
   - Timeout: 240,000 ms (4 minutes).  
   - Body (JSON):  
     ```json
     {
       "model": "jina-deepsearch-v1",
       "messages": [{
         "role": "user",
         "content": "I'm looking to find sale leads.\nI need to find the decision maker for the business... Business_Name: {{ $json.Company_name }}\nWebsite: {{ $json.Company_website }}\n... Lead_Name, role, email and a quick summary..."
       }],
       "stream": false,
       "reasoning_effort": "low"
     }
     ```  
   - Connect "Loop Over Items" → "Lead Person Research".

7. **Merge Research Output**  
   - Add "Merge" node named "Merge-Search" with mode "combine" and option "combineAll".  
   - Connect "Lead Person Research" → "Merge-Search" (input 1).  
   - Connect "Loop Over Items" → "Merge-Search" (input 2).

8. **Lead Analyzer Agent**  
   - Add Langchain Agent node "Lead Analyzer".  
   - Use OpenAI o3-mini model (add Langchain OpenAI node "OpenAI o3-mini" connected internally).  
   - Configure prompt including goal to identify best lead from deep research and company info with output JSON schema: lead_name, lead_email, company_name, summary_of_your_though.  
   - Set max iterations 4, system role as Lead Generation Specialist.  
   - Connect "Merge-Search" → "Lead Analyzer".

9. **Parse Lead Analyzer Output**  
   - Add Langchain Output Parser node "Structured Output Parser".  
   - Provide JSON schema example for expected fields.  
   - Connect "Lead Analyzer" → "Structured Output Parser".

10. **Add Wait Node**  
    - Add "Wait" node "Wait-5-sec" with 5 seconds delay.  
    - Connect "Structured Output Parser" → "Wait-5-sec".

11. **Email Content Creator Agent**  
    - Add Langchain Agent node "Email Content Creator".  
    - Use OpenAI 4o-mini model with temperature 0.8 ("OpenAI - 4o-mini" node connected internally).  
    - Configure prompt to write professional, friendly marketing email and subject using business info and lead info.  
    - Output fields: email_subject, email_wrote, your_summary.  
    - Connect "Wait-5-sec" → "Email Content Creator".

12. **Parse Email Content Output**  
    - Add Langchain Output Parser "Structured Output Parser3" with JSON schema for email fields.  
    - Connect "Email Content Creator" → "Structured Output Parser3".

13. **Evaluator Agent**  
    - Add Langchain Agent node "Evaluator".  
    - Use OpenAI 4o-mini - low temperature model ("OpenAI - 4o-mini - low" node connected internally).  
    - Configure prompt to evaluate email using detailed framework: subject effectiveness, content, readability, call-to-action, technical elements.  
    - Output fields: overall_score, areas_for_improvement, your_summary.  
    - Connect "Structured Output Parser3" → "Evaluator".

14. **Parse Evaluation Output**  
    - Add Langchain Output Parser "Structured Output Parser1" with JSON schema for evaluation results.  
    - Connect "Evaluator" → "Structured Output Parser1".

15. **Merge Lead Data and Evaluation**  
    - Add "Merge" node "Merge-loop" with mode "combine" and option "combineAll".  
    - Connect "Loop Over Items" → "Merge-loop" (input 1).  
    - Connect "Structured Output Parser1" → "Merge-loop" (input 2).

16. **Update Airtable Record**  
    - Add "Airtable" node "Update record".  
    - Configure with same base and table as input records.  
    - Set operation "Update" with key `{id} = {{ $json.id }}`.  
    - Map fields:  
      - processed = "yes"  
      - lead_name = `{{ $json.output.contact_name }}`  
      - lead_email = `{{ $json.output.contact_email }}`  
      - email_subject = `{{ $json.output.email_subject }}`  
      - email_text = `{{ $json.output.email_wrote }}`  
      - email_summary = `{{ $json.output.your_summary }}`  
      - create_date = current timestamp `{{ DateTime.fromISO($now).toISO() }}`  
      - task_result = "Email wrote"  
    - Connect "Merge-loop" → "Update record".

17. **Wait Between Batches**  
    - Add "Wait" node "Wait-8-sec" with 8 seconds delay.  
    - Connect "Update record" → "Wait-8-sec".

18. **Loop Back for Next Batch**  
    - Connect "Wait-8-sec" → "Loop Over Items" to continue processing next batch.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow uses AI agents to enrich company data from Airtable, perform deep research with Jina AI, draft personalized outreach emails with OpenAI, and evaluate them before updating records. Ideal for sales and marketing teams. | Sticky Note on node "Sticky Note" (disabled) in workflow. |
| Airtable node documentation for integration setup: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/ | Sticky Notes 6 and 10 |
| HTTP Request node documentation for API calls: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/ | Sticky Note 2 |
| Langchain Agent node documentation: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/ | Sticky Notes 7, 8, 9 |
| Jina AI API key generation: https://jina.ai/ | Sticky Note 5 |
| OpenAI API credentials required for Langchain nodes. | N/A |
| Join n8n community for support: Discord https://discord.com/invite/XPKeKXeB7d and Forum https://community.n8n.io/ | Sticky Note |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.