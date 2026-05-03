Generate Personalized Cold Email Icebreakers with Perplexity Research & Claude Sonnet

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-icebreakers-with-perplexity-research---claude-sonnet-9629


# Generate Personalized Cold Email Icebreakers with Perplexity Research & Claude Sonnet

---

### 1. Workflow Overview

This workflow automates the generation of highly personalized cold email icebreakers for sales or outreach campaigns using AI-powered research and natural language generation. It targets B2B lead lists maintained in Google Sheets, focusing on leads that have not yet received an icebreaker message.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception:** Fetch leads from a Google Sheets spreadsheet, filtering for those that require icebreakers.
- **1.2 Batch Processing:** Process leads in manageable chunks (batches of 25) to optimize API usage and runtime.
- **1.3 Company Name Cleanup:** Standardize and simplify company names to sound natural and human-like in the icebreakers.
- **1.4 Company Research:** Perform real-time web research using Perplexity Sonar AI, feeding contextual info about the contact and company.
- **1.5 Icebreaker Generation:** Use Claude Sonnet AI to craft personalized, casual, and relevant cold email opening lines based on the research.
- **1.6 Output Update:** Update the source Google Sheet with the generated icebreakers and cleaned company names, matched precisely by lead ID.

Each block is connected sequentially, ensuring data flows from raw input to polished output with integrated AI at critical steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Retrieves lead data from a specified Google Sheets spreadsheet. Filters leads based on the `Icebreaker` column to only process entries marked as `"no data"` (i.e., unprocessed).

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Get Leads` (Google Sheets)  
  - `Check If Unprocessed ` (If Node)

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Configuration: Default, no parameters.  
    - Input/Output: No input; outputs trigger `Get Leads`.  
    - Edge Cases: None inherent, but manual trigger requires user action.

  - **Get Leads**  
    - Type: Google Sheets  
    - Role: Fetch all rows from the configured spreadsheet.  
    - Configuration:  
      - Document ID and Sheet Name must be set by user.  
      - No filter here; fetches all rows to be filtered downstream.  
    - Input: Trigger from manual start.  
    - Output: Emits all rows from the sheet.  
    - Edge Cases: Possible connection/auth errors with Google Sheets OAuth2; empty sheet or missing columns cause downstream issues.

  - **Check If Unprocessed**  
    - Type: If node  
    - Role: Filters leads where `Icebreaker` equals `"no data"`.  
    - Configuration: Condition on `$json.Icebreaker == "no data"`.  
    - Input: Data from `Get Leads`.  
    - Output: True branch proceeds to batch processing; false branch discards.  
    - Edge Cases: Case sensitivity matters; if column missing or null, leads are skipped.

---

#### 2.2 Batch Processing

- **Overview:**  
  Processes the filtered leads in batches of 25 to manage API calls and workflow performance.

- **Nodes Involved:**  
  - `Loop Over Leads` (SplitInBatches)

- **Node Details:**  

  - **Loop Over Leads**  
    - Type: SplitInBatches  
    - Role: Splits incoming lead data into batches of 25.  
    - Configuration: Batch size set to 25.  
    - Input: True branch output from `Check If Unprocessed`.  
    - Output: Each batch triggers downstream processing nodes.  
    - Edge Cases: If fewer than 25 leads, batch size dynamically adjusts; empty input results in no output.

---

#### 2.3 Company Name Cleanup

- **Overview:**  
  Cleans and standardizes company names for natural language use in icebreakers by removing generic suffixes and extraneous words.

- **Nodes Involved:**  
  - `Company Name Clean Up` (OpenAI GPT-4o-mini)

- **Node Details:**  

  - **Company Name Clean Up**  
    - Type: OpenAI (GPT-4o-mini) node via LangChain  
    - Role: Parses `organization_name` from each lead and returns a cleaned, human-friendly company name.  
    - Configuration:  
      - System prompt defines assistant role.  
      - User prompt instructs on cleanup rules (e.g., remove “Agency”, “Inc.”, “Group”, paraphrase company names).  
      - Output as JSON with key `CompanyNameCleanUp`.  
      - Temperature 0.6 for some creativity but consistency.  
      - Max tokens 250.  
    - Input: Batches from previous node, accessing `organization_name`.  
    - Output: JSON with cleaned company name.  
    - Edge Cases: Unexpected or malformed names may produce inconsistent outputs; retry enabled on failure.  
    - Version Specific: Requires OpenAI API key configured.  

---

#### 2.4 Company Research

- **Overview:**  
  Uses Perplexity Sonar model to perform real-time web research, enriching context with current information on the lead’s founder name, title, and company, plus date.

- **Nodes Involved:**  
  - `Perplexity Sonar` (OpenRouter Language Model)  
  - `Perplexity Research` (Chain LLM)

- **Node Details:**  

  - **Perplexity Sonar**  
    - Type: OpenRouter LLM node with model `perplexity/sonar`  
    - Role: Runs research queries to gather relevant info for icebreaker generation.  
    - Configuration: No additional options; uses input from prior node.  
    - Input: Data updated with cleaned company name.  
    - Output: Textual research data.  
    - Edge Cases: API rate limits, network errors, or missing info from source can lead to incomplete data.  

  - **Perplexity Research**  
    - Type: Chain LLM (LangChain)  
    - Role: Formats and sends research prompt with lead details and current date.  
    - Configuration:  
      - Prompt: `[first_name] [last_name] [title] [organization_name] at [current date in dd.MM.yyyy]`  
      - Uses dynamic expressions to inject data.  
    - Input: Output from `Company Name Clean Up`.  
    - Output: Research context for next node.  
    - Edge Cases: Missing data fields could affect prompt quality; date formatting must be supported by n8n version.

---

#### 2.5 Icebreaker Generation

- **Overview:**  
  Generates a personalized, casual, one-line icebreaker sentence for cold emails using Claude Sonnet AI, leveraging research data and cleaned company names.

- **Nodes Involved:**  
  - `Claude Sonnet` (OpenRouter Language Model)  
  - `Generate Icebreaker` (Chain LLM)  
  - `Parse JSON` (Output Parser Structured)

- **Node Details:**  

  - **Claude Sonnet**  
    - Type: OpenRouter LLM node with model `anthropic/claude-sonnet-4`  
    - Role: Generates human-like, nuanced icebreaker text.  
    - Configuration:  
      - TopP = 0.8, MaxTokens = 300 for balanced creativity and length.  
    - Input: Research data from previous block.  
    - Output: Draft icebreaker text.  
    - Edge Cases: Model response variability, API quotas, or timeouts.

  - **Generate Icebreaker**  
    - Type: Chain LLM with integrated output parser  
    - Role: Defines multi-step prompt including:  
      - Friendly tone ("casual bar conversation")  
      - Use of provided research context  
      - Instructions to not fabricate information  
      - Specific template formatting including greeting and closing phrase with cleaned company name  
    - Configuration:  
      - Uses prompt templates with embedded expressions referencing cleaned company name and research text.  
      - Output parsed as structured JSON with key `icebreaker`.  
    - Input: Output from `Claude Sonnet`.  
    - Output: Parsed icebreaker JSON.  
    - Edge Cases: Parsing errors if AI output is malformed; prompt failures if data missing.

  - **Parse JSON**  
    - Type: Output Parser Structured  
    - Role: Validates and extracts structured JSON from AI output.  
    - Configuration: JSON schema example provided for `icebreaker` field.  
    - Input: AI-generated text from `Claude Sonnet`.  
    - Output: Structured JSON for downstream use.  
    - Edge Cases: Malformed JSON leading to errors; fallback handling required.

---

#### 2.6 Output Update

- **Overview:**  
  Updates the original Google Sheets lead data by inserting the cleaned company name and the generated icebreaker text, matching rows by unique `id`.

- **Nodes Involved:**  
  - `Update Icebreaker` (Google Sheets)

- **Node Details:**  

  - **Update Icebreaker**  
    - Type: Google Sheets  
    - Role: Updates rows where `id` matches, setting `Icebreaker` and `company_name_cleanup` columns.  
    - Configuration:  
      - Document ID and Sheet Name must be set.  
      - Matching column set to `id` for accurate update.  
      - Mapping mode set to define columns explicitly.  
    - Input: Output from `Generate Icebreaker`.  
    - Output: Confirmation of update (not further used).  
    - Edge Cases: Update failures if `id` mismatch; Google Sheets API errors; concurrency issues if the sheet is edited simultaneously.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                         | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                   |
|-------------------------|--------------------------------------|---------------------------------------|----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Starts workflow execution manually     | —                          | Get Leads                   |                                                                                                               |
| Get Leads               | Google Sheets                        | Fetch all leads from spreadsheet       | When clicking ‘Execute workflow’ | Check If Unprocessed         | ## 📊 DATA SOURCE: GOOGLE SHEETS<br>Your spreadsheet must have columns: `id`, `first_name`, `last_name`, `title`, `organization_name`, `Icebreaker`, `company_name_cleanup`. Only rows with `Icebreaker` = "no data" are processed. |
| Check If Unprocessed     | If Node                             | Filters leads needing icebreakers      | Get Leads                  | Loop Over Leads             |                                                                                                               |
| Loop Over Leads          | SplitInBatches                      | Batch processing (25 leads)             | Check If Unprocessed        | Company Name Clean Up       | ## 🔄 PROCESSING<br>Batch size: 25 leads at a time                                                            |
| Company Name Clean Up    | OpenAI GPT-4o-mini (LangChain)      | Standardizes company names              | Loop Over Leads             | Perplexity Research         | ## ✏️ COMPANY NAME STANDARDIZATION<br>Cleans company names to sound natural in icebreakers (e.g., remove “Agency”). |
| Perplexity Sonar         | OpenRouter LLM (perplexity/sonar)   | Performs real-time company research    | Company Name Clean Up       | Perplexity Research         | ## 🔎 COMPANY RESEARCH<br>Perplexity Sonar conducts web research on founder name, title, company, and date.    |
| Perplexity Research      | Chain LLM (LangChain)                | Formats research prompt                 | Perplexity Sonar            | Claude Sonnet               |                                                                                                               |
| Claude Sonnet            | OpenRouter LLM (anthropic/claude-sonnet-4) | Generates nuanced, human-like text     | Perplexity Research         | Generate Icebreaker         | ## ✨ ICEBREAKER GENERATION<br>Claude Sonnet crafts personalized opening lines with human-like nuance.          |
| Parse JSON              | Output Parser Structured             | Extracts structured JSON from AI output| Claude Sonnet               | Generate Icebreaker         |                                                                                                               |
| Generate Icebreaker      | Chain LLM (LangChain)                | Creates final icebreaker text           | Parse JSON / Claude Sonnet  | Update Icebreaker           |                                                                                                               |
| Update Icebreaker        | Google Sheets                       | Updates spreadsheet with icebreaker    | Generate Icebreaker         | Loop Over Leads             | ## 💾 UPDATE SPREADSHEET<br>Updates `company_name_cleanup` and `Icebreaker` fields matched by `id`.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add Google Sheets Node to Get Leads:**  
   - Name: `Get Leads`  
   - Operation: Read rows  
   - Set your spreadsheet Document ID and Sheet Name (must match your leads sheet).  
   - Connect output of Manual Trigger to this node.

3. **Add If Node to Filter Leads:**  
   - Name: `Check If Unprocessed`  
   - Condition:  
     - `$json.Icebreaker` equals `"no data"` (case-sensitive).  
   - Connect `Get Leads` output to this node.

4. **Add SplitInBatches Node:**  
   - Name: `Loop Over Leads`  
   - Batch size: 25  
   - Connect True output of `Check If Unprocessed` to this node.

5. **Add OpenAI Node for Company Name Cleanup:**  
   - Name: `Company Name Clean Up`  
   - Credentials: OpenAI API key (GPT-4o-mini)  
   - Model: `gpt-4o-mini`  
   - Temperature: 0.6; Max Tokens: 250  
   - Prompt (system): "You are a helpful, intelligent writing assistant."  
   - Prompt (user): Instructions to clean company names by removing words like "Agency", "Inc.", "Group" and paraphrasing to sound natural. Return JSON with key `CompanyNameCleanUp`.  
   - Connect output of `Loop Over Leads` to this node.

6. **Add OpenRouter LLM Node for Perplexity Sonar Research:**  
   - Name: `Perplexity Sonar`  
   - Model: `perplexity/sonar`  
   - Credentials: OpenRouter API (with Perplexity access)  
   - Connect output of `Company Name Clean Up` node here.

7. **Add Chain LLM Node for Perplexity Research Prompt:**  
   - Name: `Perplexity Research`  
   - Model: Any (or as per workflow; this node formats research prompt)  
   - Prompt: Combine lead info: `{{first_name}} {{last_name}} {{title}} {{organization_name}} at {{current_date (dd.MM.yyyy)}}`  
   - Connect output of `Perplexity Sonar` to this node.

8. **Add OpenRouter LLM Node for Claude Sonnet Icebreaker Generation:**  
   - Name: `Claude Sonnet`  
   - Model: `anthropic/claude-sonnet-4`  
   - Parameters: TopP=0.8, MaxTokens=300  
   - Credentials: OpenRouter API (with Claude Sonnet access)  
   - Connect output of `Perplexity Research` node.

9. **Add Output Parser Structured Node:**  
   - Name: `Parse JSON`  
   - Schema example: `{"icebreaker":""}`  
   - Connect output of `Claude Sonnet` node.

10. **Add Chain LLM Node to Generate Icebreaker Text:**  
    - Name: `Generate Icebreaker`  
    - Model: GPT-4o-mini or LangChain Chain LLM  
    - Prompt instructions:  
      - Write a casual, human-like icebreaker line for cold emails.  
      - Use cleaned company name and Perplexity research context.  
      - Follow formatting rules (greeting with first name, casual tone, no fabrications).  
      - End with "I wanted to share something that could be relevant for [Cleaned Company Name]".  
    - Enable output parsing.  
    - Connect both `Parse JSON` and `Claude Sonnet` outputs to this node as per workflow or configure as necessary.

11. **Add Google Sheets Node to Update Icebreaker:**  
    - Name: `Update Icebreaker`  
    - Operation: Update  
    - Document ID and Sheet Name: same as `Get Leads` but the sheet must be writable.  
    - Matching column: `id`  
    - Columns to update: `Icebreaker` (with generated icebreaker), `company_name_cleanup` (with cleaned company name)  
    - Connect output of `Generate Icebreaker` node.

12. **Connect output of `Update Icebreaker` to continue looping:**  
    - Connect back to `Loop Over Leads` node to process next batch until all leads are processed.

13. **Configure Credentials:**  
    - Google Sheets OAuth2 for read/write access  
    - OpenAI API key for GPT-4o-mini (Company Name Cleanup, Generate Icebreaker)  
    - OpenRouter API key with access to Perplexity Sonar and Claude Sonnet models

14. **Test:**  
    - Run with a small subset of leads (2-3) to verify processing and correct output.  
    - Confirm Google Sheet updates correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Quick Start Guide: Prepare Google Sheets with required columns, configure OpenAI and OpenRouter credentials, test with few leads, then run full workflow.                                    | Sticky Note “Quick Start” in workflow                         |
| Cost Analysis: Estimated cost per lead is approximately $0.02; includes GPT-4o-mini, Perplexity Sonar, and Claude Sonnet API usage. Typical ROI can be 100-1000x if response rate improves. | Sticky Note “Cost Analysis” in workflow                       |
| Data Source Requirements: Google Sheets must have columns `id`, `first_name`, `last_name`, `title`, `organization_name`, `Icebreaker` (set to "no data" for new leads), and `company_name_cleanup`. | Sticky Note “Data Source” in workflow                         |
| Company Name Cleanup Examples: Simplifies company names by removing generic suffixes (e.g., “Agency”, “Inc.”) to sound more natural in outreach.                                             | Sticky Note “Company Name Cleanup” in workflow                |
| Perplexity Research Example Prompt: Combines founder name, job title, company, and current date in dd.MM.yyyy format for real-time research context.                                         | Sticky Note “Research” in workflow                            |
| Claude Sonnet Advantages: Generates nuanced, human-like, context-aware icebreakers superior to generic AI outputs.                                                                            | Sticky Note “Icebreaker Generation” in workflow               |

---

**Disclaimer:** The provided text and analysis originate exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.

---