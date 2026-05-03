Generate AI Search Visibility Datasets with Claude and GPT for Tracking Platforms

https://n8nworkflows.xyz/workflows/generate-ai-search-visibility-datasets-with-claude-and-gpt-for-tracking-platforms-11748


# Generate AI Search Visibility Datasets with Claude and GPT for Tracking Platforms

### 1. Workflow Overview

This workflow, titled **"Generate AI Search Visibility Datasets with Claude and GPT for Tracking Platforms"**, automates the creation of tailored AI search prompt datasets for monitoring online visibility of companies within specific industries and categories. It is designed primarily for SEO/GEO marketing teams, Growth Managers, GTM engineers, and founders who want to generate comprehensive, industry-specific search queries and prompts that can be used in AI visibility tracking platforms such as ALLMO.ai.

The workflow logically divides into three main functional blocks:

- **1.1 Input Reception and Company Data Preparation**: Capture user input (company name and website) and prepare this data for research.
- **1.2 AI-Powered Company Research and Data Extraction**: Use GPT-5 Mini to conduct detailed company research by visiting the given website and extracting structured company information.
- **1.3 Prompt Dataset Generation and Processing**: Generate varied AI search prompt datasets using Claude and GPT models, including query refinement, keyword generation, and natural question formulation.
- **1.4 Optional Translation and Final Dataset Preparation**: Translate the English dataset into German (optional), merge datasets, add metadata, and prepare the final dataset for export as a file ready for upload to visibility tracking tools.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Company Data Preparation

**Overview:**  
This block receives user inputs for the company name and website URL via a form submission node and prepares the input data fields for downstream processing.

**Nodes Involved:**  
- On form submission  
- Edit Fields  

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Collect user input for company name and website URL through a web form.  
  - Configuration: Two form fields — "Provide a Company Name" and "Provide a Website URL" with placeholders.  
  - Inputs: User web form submission  
  - Outputs: JSON object with user inputs  
  - Failure modes: User cancellation, invalid or missing inputs  
  - Sticky Note: Explains the trigger purpose and alternative data sources (e.g., API, Google Sheets)

- **Edit Fields**  
  - Type: Set Node  
  - Role: Rename and assign user input JSON fields to consistent variable names (`company_name`, `company_website`).  
  - Configuration: Assign `company_name` from form input "Provide a Company Name"; assign `company_website` from "Provide a Website URL".  
  - Inputs: Output from form submission node  
  - Outputs: JSON with standardized keys  
  - Edge cases: Missing or malformed input data

---

#### Block 1.2: AI-Powered Company Research and Data Extraction

**Overview:**  
This block leverages GPT-5 Mini to perform web-based research on the provided company website to extract detailed company profile data including industry, product category, buyer personas, problems solved, key features, and value proposition.

**Nodes Involved:**  
- GPT_company_research  
- Code in JavaScript  
- Remove citations  
- Edit Fields2  

**Node Details:**  

- **GPT_company_research**  
  - Type: HTTP Request (OpenAI API)  
  - Role: Sends a complex system prompt to GPT-5 Mini instructing it to research the company website and return structured JSON data containing company profile fields.  
  - Configuration: POST request with JSON body including system and user messages, tool usage (web search filtered to company domain), and tool choice set to auto.  
  - Inputs: JSON with `company_name` and `company_website`  
  - Outputs: Raw GPT response containing JSON in text form  
  - Potential failures: API auth errors, timeouts (timeout set to 25 minutes), malformed responses, website inaccessible  

- **Code in JavaScript** (after GPT_company_research)  
  - Type: Code  
  - Role: Extracts and parses JSON string from GPT response to structured JSON object.  
  - Inputs: Raw GPT response  
  - Outputs: Parsed JSON object with company profile data  
  - Edge cases: Parsing errors, unexpected response format  

- **Remove citations**  
  - Type: Code  
  - Role: Recursively cleans all string fields in the JSON object to remove citation patterns (e.g., markdown links or references) from text fields.  
  - Inputs: Parsed company profile JSON  
  - Outputs: Cleaned JSON data  
  - Potential failures: Unexpected data types, regex failures  

- **Edit Fields2**  
  - Type: Set Node  
  - Role: Flattens the nested JSON structure into individual fields with descriptive names for later use in prompt generation.  
  - Configuration: Assigns fields such as `industry`, `Unbranded category`, `category`, buyer persona names and descriptions, problems, key features, and value proposition into separate variables.  
  - Inputs: Cleaned JSON object  
  - Outputs: Flattened JSON with explicit fields for each data point  
  - Edge cases: Missing or incomplete data fields  

- Sticky Note: Explains this block is for company research and extraction of marketing relevant dimensions.

---

#### Block 1.3: Prompt Dataset Generation and Processing

**Overview:**  
Generates a diverse set of AI search prompts by combining company-specific data with AI-generated phrases and keywords. This block uses Claude and GPT models to create refined phrases, typical search terms, and natural human-like questions.

**Nodes Involved:**  
- Hard coded phrase - returns 13  
- Claude_text_writer  
- Code in JavaScript2 (parses Claude JSONL output)  
- GPT_keywords_return 5  
- Code in JavaScript1 (parses GPT keywords output)  
- Claude_text_writer_return7  
- Code in JavaScript3 (parses Claude questions output)  
- Merge  
- Clean-up  
- Merge1  
- English full dataset  

**Node Details:**  

- **Hard coded phrase - returns 13**  
  - Type: Code  
  - Role: Builds 13 hardcoded phrase templates combining company data fields (industry, category, features, problems) into search query phrases.  
  - Inputs: Flattened fields from Edit Fields2 node  
  - Outputs: JSON array of 13 phrases  
  - Edge cases: Missing data leads to incomplete phrases  

- **Claude_text_writer**  
  - Type: Anthropic (Claude) Langchain node  
  - Role: Refines and lightly improves the 13 phrases to sound natural and human-like, preserving meaning and formatting the output as JSONL.  
  - Inputs: 13 phrases in JSONL format from previous node  
  - Outputs: JSONL with refined phrases  
  - Edge cases: Model response issues, JSONL format errors  

- **Code in JavaScript2**  
  - Type: Code  
  - Role: Parses the JSONL output from Claude_text_writer to extract the array of refined phrases.  
  - Inputs: Raw Claude JSONL output  
  - Outputs: Array of phrases  
  - Edge cases: Parsing errors, unexpected output  

- **GPT_keywords_return 5**  
  - Type: HTTP Request (OpenAI API)  
  - Role: Requests GPT-5 Mini to generate 5 top non-brand search terms relevant to the company based on industry and category.  
  - Inputs: Company name, website, industry, category  
  - Outputs: Raw GPT output with 5 search terms in JSON  
  - Edge cases: API failures, missing inputs, brand name leakage  

- **Code in JavaScript1**  
  - Type: Code  
  - Role: Extracts and parses the 5 search terms from the GPT output JSON.  
  - Inputs: Raw GPT output  
  - Outputs: Array of 5 search terms  
  - Edge cases: Parsing errors  

- **Claude_text_writer_return7**  
  - Type: Anthropic (Claude) Langchain node  
  - Role: Generates 7 natural, unbranded AI search prompts/questions by combining company data elements with specific question templates that simulate real user queries.  
  - Inputs: Flattened company profile JSON from Edit Fields2  
  - Outputs: JSON array of 7 AI-generated questions  
  - Edge cases: Model output format issues, branding leakage  

- **Code in JavaScript3**  
  - Type: Code  
  - Role: Extracts the 7 questions from Claude output JSON.  
  - Inputs: Raw Claude JSON output  
  - Outputs: Array of 7 prompts/questions  
  - Edge cases: Parsing failures  

- **Merge**  
  - Type: Merge (Combine)  
  - Role: Merges the outputs of refined phrases and GPT keywords to consolidate into one dataset branch.  
  - Inputs: Phrases and keywords arrays  
  - Outputs: Combined array of phrases and keywords  

- **Clean-up**  
  - Type: Code  
  - Role: Combines all collected phrases and keywords into a single list for uniform further processing.  
  - Inputs: Merged arrays from Merge node  
  - Outputs: Consolidated prompts array with language and country metadata  

- **Merge1**  
  - Type: Merge  
  - Role: Merges the cleaned prompt list with the AI-generated natural questions (from Claude_text_writer_return7) for a full English dataset.  
  - Inputs: Cleaned prompts and Claude questions  
  - Outputs: Full English prompt dataset  

- **English full dataset**  
  - Type: Code  
  - Role: Combines all English prompts and questions into a single JSON array with language and country metadata.  
  - Inputs: Outputs from Merge1  
  - Outputs: Final English prompts array ready for translation or export  

- Sticky Note: Describes this block as the core prompt dataset creation phase using hardcoded, GPT, and Claude generation methods for variety.

---

#### Block 1.4: Optional Translation and Final Dataset Preparation

**Overview:**  
Optionally translates the English prompts into German, merges datasets, adds metadata fields, and prepares the final dataset for export as a CSV file ready for upload to AI search visibility tools.

**Nodes Involved:**  
- Claude_text_writer-first12  
- Code in JavaScript4  
- Merge2  
- Set required fields for upload  
- Convert to File  

**Node Details:**  

- **Claude_text_writer-first12**  
  - Type: Anthropic (Claude) Langchain node  
  - Role: Translates the full English prompt dataset (25 phrases/questions) into German, preserving JSON structure and phrase count.  
  - Inputs: English prompts array  
  - Outputs: JSON with `translated_phrases` array in German  
  - Edge cases: Translation quality, output format  

- **Code in JavaScript4**  
  - Type: Code  
  - Role: Extracts the translated German phrases array from Claude's output JSON and maps each phrase into individual JSON items with language and country codes.  
  - Inputs: Raw Claude German translation output  
  - Outputs: Array of German prompt JSON objects with metadata (language "german", country "DE")  
  - Edge cases: Parsing failures  

- **Merge2**  
  - Type: Merge  
  - Role: Merges English and German datasets into one combined dataset for export.  
  - Inputs: English and German prompt arrays  
  - Outputs: Combined bilingual prompt dataset  

- **Set required fields for upload**  
  - Type: Code  
  - Role: Adds necessary metadata fields to each prompt item for upload into AI visibility tracking tools including model names, tracking interval, status, and search question mapping.  
  - Inputs: Combined prompt dataset  
  - Outputs: Enriched prompt dataset with upload-ready fields  

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts the final enriched dataset into a file format (CSV or JSON) suitable for download or direct upload to external platforms.  
  - Inputs: Final dataset with metadata  
  - Outputs: File for export/download  

- Sticky Notes: Describe optional translation path and configuration for direct upload readiness.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                                          | Input Node(s)                    | Output Node(s)              | Sticky Note                                                                                               |
|-----------------------------|-----------------------------------|----------------------------------------------------------|---------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------|
| On form submission           | Form Trigger                      | Collect company name and website from user input         | (Trigger)                       | Edit Fields                 | Provide a Company Name and Website URL. You can also connect this to an API, Google Sheets or other data source. |
| Edit Fields                 | Set                               | Normalize input fields into `company_name` and `company_website` | On form submission              | GPT_company_research        |                                                                                                         |
| GPT_company_research         | HTTP Request (OpenAI GPT-5 Mini)  | Research company using website, extract structured profile | Edit Fields                    | Code in JavaScript          | Company Research: extracts key business model and marketing profile information for prompt dataset creation.|
| Code in JavaScript           | Code                              | Parse GPT response JSON string into structured object    | GPT_company_research            | Remove citations            |                                                                                                         |
| Remove citations             | Code                              | Clean citation patterns from all strings in JSON data    | Code in JavaScript              | Edit Fields2                |                                                                                                         |
| Edit Fields2                | Set                               | Flatten company profile JSON into explicit named fields  | Remove citations               | Hard coded phrase - returns 13, GPT_keywords_return 5, Claude_text_writer_return7 |                                                                                                         |
| Hard coded phrase - returns 13 | Code                            | Create 13 templated search phrases combining company data | Edit Fields2                   | Claude_text_writer          |                                                                                                         |
| Claude_text_writer           | Anthropic Claude Langchain node   | Refine 13 phrases into natural, human-like search queries | Hard coded phrase - returns 13 | Code in JavaScript2         |                                                                                                         |
| Code in JavaScript2          | Code                              | Parse Claude JSONL output to extract refined phrases     | Claude_text_writer             | Merge                      |                                                                                                         |
| GPT_keywords_return 5        | HTTP Request (OpenAI GPT-5 Mini)  | Generate 5 top non-brand search terms based on company profile | Edit Fields2                  | Code in JavaScript1         |                                                                                                         |
| Code in JavaScript1          | Code                              | Parse GPT output JSON to extract search terms            | GPT_keywords_return 5          | Merge                      |                                                                                                         |
| Claude_text_writer_return7   | Anthropic Claude Langchain node   | Generate 7 natural unbranded AI search questions         | Edit Fields2                  | Code in JavaScript3         |                                                                                                         |
| Code in JavaScript3          | Code                              | Parse Claude JSON output to extract questions             | Claude_text_writer_return7     | Merge1                     |                                                                                                         |
| Merge                       | Merge                             | Combine refined phrases and GPT keywords                  | Code in JavaScript2, Code in JavaScript1 | Clean-up                   |                                                                                                         |
| Clean-up                    | Code                              | Consolidate all phrases and keywords into one list        | Merge                         | Merge1                     |                                                                                                         |
| Merge1                      | Merge                             | Merge cleaned phrases with natural questions              | Clean-up, Code in JavaScript3  | English full dataset        |                                                                                                         |
| English full dataset         | Code                              | Combine all English prompts and questions into one array  | Merge1                        | Turn into table, Claude_text_writer-first12 |                                                                                                         |
| Claude_text_writer-first12   | Anthropic Claude Langchain node   | Translate English prompt dataset into German              | English full dataset           | Code in JavaScript4         | Optional: Translation into any language. Preset: German. Remove to continue without translation.          |
| Code in JavaScript4          | Code                              | Extract translated German phrases and add metadata        | Claude_text_writer-first12     | Merge2                     |                                                                                                         |
| Merge2                      | Merge                             | Combine English and German datasets                        | Turn into table, Code in JavaScript4 | Set required fields for upload |                                                                                                         |
| Set required fields for upload | Code                            | Add metadata for AI visibility tools upload readiness     | Merge2                        | Convert to File             | Optional: Configure dataset for upload with model info, tracking interval, and status                     |
| Convert to File              | Convert to File                   | Export final dataset as file (CSV/JSON)                   | Set required fields for upload |                             |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure with form title "Input" and two fields: "Provide a Company Name" (placeholder: ALLMO.ai) and "Provide a Website URL" (placeholder: www.allmo.ai).

2. **Create Set Node "Edit Fields"**  
   - Assign `company_name` from form field "Provide a Company Name".  
   - Assign `company_website` from form field "Provide a Website URL".  
   - Connect output from Form Trigger node.

3. **Create HTTP Request Node "GPT_company_research"**  
   - Use OpenAI API credentials with GPT-5 Mini.  
   - POST to https://api.openai.com/v1/responses  
   - Set JSON body with system and user prompts instructing to research the provided company and website, specifying strict output JSON schema (industry, solution, buyer personas, problems, features, value proposition).  
   - Enable web search tool filtered to the company domain.  
   - Connect input from "Edit Fields".

4. **Create Code Node "Code in JavaScript"**  
   - Parse the GPT response output text to extract the JSON object.  
   - Connect input from "GPT_company_research".

5. **Create Code Node "Remove citations"**  
   - Implement recursive cleaning of citation-like patterns in all string fields of the JSON object.  
   - Connect input from "Code in JavaScript".

6. **Create Set Node "Edit Fields2"**  
   - Map nested JSON fields to flat named variables (industry, unbranded category, category, persona names/descriptions, problems, features, value proposition).  
   - Connect input from "Remove citations".

7. **Create Code Node "Hard coded phrase - returns 13"**  
   - Construct 13 search phrase templates combining the company data fields into natural language phrases.  
   - Connect input from "Edit Fields2".

8. **Create Anthropic Claude Node "Claude_text_writer"**  
   - Model: claude-sonnet-4-5-20250929  
   - Task: Refine 13 input phrases, ensure natural phrasing without changing meaning, output JSONL.  
   - Connect input from "Hard coded phrase - returns 13".  
   - Add Anthropic API credentials.

9. **Create Code Node "Code in JavaScript2"**  
   - Parse Claude JSONL output into an array of phrases.  
   - Connect input from "Claude_text_writer".

10. **Create HTTP Request Node "GPT_keywords_return 5"**  
    - Use OpenAI GPT-5 Mini API.  
    - Input: company name, website, industry, category.  
    - Task: Generate 5 top non-brand search terms.  
    - Connect input from "Edit Fields2".  
    - Add OpenAI API credentials.

11. **Create Code Node "Code in JavaScript1"**  
    - Parse GPT keywords output JSON, extract search terms array.  
    - Connect input from "GPT_keywords_return 5".

12. **Create Anthropic Claude Node "Claude_text_writer_return7"**  
    - Model: claude-sonnet-4-5-20250929  
    - Task: Generate 7 natural, unbranded AI search questions from company data fields using predefined templates.  
    - Connect input from "Edit Fields2".  
    - Add Anthropic API credentials.

13. **Create Code Node "Code in JavaScript3"**  
    - Parse Claude output JSON to extract the 7 generated questions.  
    - Connect input from "Claude_text_writer_return7".

14. **Create Merge Node "Merge"**  
    - Merge: combine arrays from "Code in JavaScript2" (refined phrases) and "Code in JavaScript1" (GPT keywords).  
    - Connect both inputs accordingly.

15. **Create Code Node "Clean-up"**  
    - Consolidate merged phrases and keywords into a single array with language "english" and country "global".  
    - Connect input from "Merge".

16. **Create Merge Node "Merge1"**  
    - Merge: combine cleaned phrases (from "Clean-up") and questions (from "Code in JavaScript3").  
    - Connect inputs appropriately.

17. **Create Code Node "English full dataset"**  
    - Combine all English prompts and questions into a single array with language and country metadata.  
    - Connect input from "Merge1".

18. **Optional: Create Anthropic Claude Node "Claude_text_writer-first12"**  
    - Translate the 25 English prompts/questions into German, preserving structure and order.  
    - Connect input from "English full dataset".  
    - Add Anthropic API credentials.

19. **Optional: Create Code Node "Code in JavaScript4"**  
    - Parse German translation output JSON, extract translated phrases array, assign language "german" and country "DE".  
    - Connect input from "Claude_text_writer-first12".

20. **Create Merge Node "Merge2"**  
    - Merge English dataset (from "Turn into table") and German dataset (from "Code in JavaScript4").  
    - Connect respective inputs.

21. **Create Code Node "Set required fields for upload"**  
    - Add metadata fields to each item: model array, tracking interval "Weekly", status "active", and map `search_question` to prompt text.  
    - Connect input from "Merge2".

22. **Create Convert to File Node "Convert to File"**  
    - Convert final dataset to CSV or JSON file for download or direct upload to tracking platforms.  
    - Connect input from "Set required fields for upload".

23. **Create Code Node "Turn into table"**  
    - Split English full dataset into individual JSON items for merging and export.  
    - Connect input from "English full dataset".

24. **Set up credentials**  
    - Anthropic API credential for Claude nodes  
    - OpenAI API credential for GPT nodes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow generates 50 AI search prompts per company: 25 English plus 25 German translations, designed for AI Search Visibility platforms like ALLMO.ai.                                                                        | Workflow general summary and use case.             |
| The workflow requires valid API credentials for Anthropic (Claude) and OpenAI (GPT) to operate.                                                                                                                                      | API credential setup instructions.                  |
| The company research step strictly requires a publicly accessible website URL to perform accurate, source-verified data extraction.                                                                                                | Data input requirements.                            |
| Prompt generation uses a combination of hardcoded templates, GPT-generated keywords, and Claude-generated natural questions to maximize variety and relevance.                                                                     | Prompt dataset creation explanation.                |
| Translation is optional and preset to German; it can be removed if only English prompts are desired.                                                                                                                                 | Translation block note.                             |
| The final output file can be directly uploaded to AI Search Visibility tools, with metadata fields prefilled for easy ingestion.                                                                                                    | Export and upload configuration note.              |
| For more on ALLMO.ai and AI search visibility tracking, visit: https://www.allmo.ai                                                                                                                                                  | Relevant platform link.                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It fully complies with content policies and contains no illegal or offensive elements. All processed data is legal and publicly accessible.